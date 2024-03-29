# TypeScript 5.4

## 从最后一次赋值以后，在闭包中保留类型细化

TypeScript 通常可以根据您进行的检查来确定变量更具体的类型。
这个过程被称为类型细化。

```ts
function uppercaseStrings(x: string | number) {
  if (typeof x === 'string') {
    // TypeScript 知道 'x' 是 'string' 类型
    return x.toUpperCase();
  }
}
```

一个常见的痛点是被细化的类型不总会在闭包函数中保留。

```ts
function getUrls(url: string | URL, names: string[]) {
  if (typeof url === 'string') {
    url = new URL(url);
  }

  return names.map(name => {
    url.searchParams.set('name', name);
    //  ~~~~~~~~~~~~
    // error!
    // Property 'searchParams' does not exist on type 'string | URL'.

    return url.toString();
  });
}
```

在这里，TypeScript 决定在我们的回调函数中不“安全”地假设 `url` *实际上*是一个 `URL` 对象，因为它在其他地方发生了变化；
然而，在这种情况下，箭头函数总是在对 `url` 的赋值之后创建的，并且它也是对 `url` 的最后一次赋值。

TypeScript 5.4 利用这一点使类型细化变得更加智能。
当在非[提升](https://developer.mozilla.org/en-US/docs/Glossary/Hoisting)的函数中使用参数和 `let` 变量时，类型检查器将寻找最后一次赋值点。
如果找到了这样的点，TypeScript 可以安全地从包含函数的外部进行类型细化。
这意味着上面的例子现在可以正常工作了。

请注意，如果变量在嵌套函数的任何地方被赋值，类型细化分析将不会生效。
这是因为无法确定该函数是否会在后续被调用。

```ts
function printValueLater(value: string | undefined) {
  if (value === undefined) {
    value = 'missing!';
  }

  setTimeout(() => {
    // Modifying 'value', even in a way that shouldn't affect
    // its type, will invalidate type refinements in closures.
    value = value;
  }, 500);

  setTimeout(() => {
    console.log(value.toUpperCase());
    //          ~~~~~
    // error! 'value' is possibly 'undefined'.
  }, 1000);
}
```

这将使许多典型的 JavaScript 代码更容易表达出来。
更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/56908)。

## `NoInfer` 工具类型

当调用泛型函数时，TypeScript 能够从实际参数推断出类型参数的值。

```ts
function doSomething<T>(arg: T) {
  // ...
}

// We can explicitly say that 'T' should be 'string'.
doSomething<string>('hello!');

// We can also just let the type of 'T' get inferred.
doSomething('hello!');
```

然而，一个挑战是并不总能够清楚推断出“最佳”的类型是什么。
这可能导致 TypeScript 拒绝合理的调用，接受有问题的调用，或者在捕捉到 bug 时报告较差的错误消息。

例如，假设 `createStreetLight` 函数接收一系列颜色名，以及一个默认颜色名。

```ts
function createStreetLight<C extends string>(colors: C[], defaultColor?: C) {
  // ...
}

createStreetLight(['red', 'yellow', 'green'], 'red');
```

当我们传入的 `defaultColor` 不在 `colors` 数组里会发生什么？
在这个函数中，`colors` 被当成“事实来源”，并描述了可以传递给 `defaultColor`。

```ts
// Oops! This undesirable, but is allowed!
createStreetLight(['red', 'yellow', 'green'], 'blue');
```

在这个调用中，类型推断决定 `"blue"` 与 `"red"`、`"yellow"` 或 `"green"` 一样有效。
因此，TypeScript 推断 `C` 的类型为 `"red" | "yellow" | "green" | "blue"`。
可以说推断结果让我们感到十分惊讶！

目前人们处理这个问题的一种方式是添加一个独立的类型参数，该参数受现有类型参数的限制。

```ts
function createStreetLight<C extends string, D extends C>(
  colors: C[],
  defaultColor?: D
) {}

createStreetLight(['red', 'yellow', 'green'], 'blue');
//                                            ~~~~~~
// error!
// Argument of type '"blue"' is not assignable to parameter of type '"red" | "yellow" | "green" | undefined'.
```

这种方法可以解决问题，但有点尴尬，因为在 `createStreetLight` 的签名中可能不会在其他地方使用 `D`。
虽然这种情况不算糟糕，但在签名中只使用一次类型参数通常是一种代码坏味道。

这就是为什么 TypeScript 5.4 引入了一个新的 `NoInfer<T>` 实用类型。
将一个类型包裹在 `NoInfer<...>` 中向 TypeScript 发出一个信号，告诉它不要深入匹配内部类型以寻找类型推断的候选项。

使用 `NoInfer`，我们可以将 `createStreetLight` 重写为以下形式：

```ts
function createStreetLight<C extends string>(
  colors: C[],
  defaultColor?: NoInfer<C>
) {
  // ...
}

createStreetLight(['red', 'yellow', 'green'], 'blue');
//                                            ~~~~~~
// error!
// Argument of type '"blue"' is not assignable to parameter of type '"red" | "yellow" | "green" | undefined'.
```

排除对 `defaultColor` 类型进行推断的探索意味着 `"blue"` 永远不会成为推断的候选项，类型检查器可以拒绝它。

具体实现请参考 [PR](https://github.com/microsoft/TypeScript/pull/56794)，以及最初实现 [PR](https://github.com/microsoft/TypeScript/pull/52968)，感谢[Mateusz Burzyński](https://github.com/Andarist)

## `Object.groupBy` 和 `Map.groupBy`

TypeScript 5.4 为 JavaScript 的新静态方法 `Object.groupBy` 和 `Map.groupBy` 添加了声明。

`Object.groupBy` 接受一个可迭代对象和一个函数，该函数确定每个元素应该被放置在哪个“组”中。
该函数需要为每个不同的分组生成一个“键”，而 `Object.groupBy` 使用该键来创建一个对象，其中每个键都映射到一个包含原始元素的数组。

因此：

```js
const array = [0, 1, 2, 3, 4, 5];

const myObj = Object.groupBy(array, (num, index) => {
  return num % 2 === 0 ? 'even' : 'odd';
});
```

等同于：

```js
const myObj = {
  even: [0, 2, 4],
  odd: [1, 3, 5],
};
```

`Map.groupBy` 类似，但生成的是一个 `Map` 而不是普通对象。
如果您需要 `Map` 提供的保证、处理期望 `Map` 的 API，或者需要使用任何类型的键进行分组（而不仅仅是可以作为 JavaScript 属性名的键），那么这可能更可取。

```js
const myObj = Map.groupBy(array, (num, index) => {
  return num % 2 === 0 ? 'even' : 'odd';
});
```

就像之前一样，你可以用等效的方式创建 `myObj`：

```js
const myObj = new Map();

myObj.set('even', [0, 2, 4]);
myObj.set('odd', [1, 3, 5]);
```

注意在 `Object.groupBy` 的例子中，生成的对象使用了所有可选属性。

```js
interface EvenOdds {
    even?: number[];
    odd?: number[];
}

const myObj: EvenOdds = Object.groupBy(...);

myObj.even;
//    ~~~~
// Error to access this under 'strictNullChecks'.
```

这是因为没法保证 `groupBy` 生成了*所有的*键。

注意这些方法仅在将 `target` 设置为 `esnext`，或者设置了相应的 `lib` 选项时才可用。
我们预计它们最终会在稳定的 `es2024` 目标下可用。

感谢[Kevin Gibbons](https://github.com/bakkot)的[PR](https://github.com/microsoft/TypeScript/pull/56805)。

## 支持在 `--moduleResolution bundler` 和 `--module preserve` 时 使用 `require()`

TypeScript 有一个名为 `bundler` 的 `moduleResolution` 选项，旨在模拟现代打包工具确定导入路径所指向的文件的方式。
该选项的一个限制是它必须与 `--module esnext` 配对使用，这导致无法使用 `import ... = require(...)` 语法。

```ts
// previously errored
import myModule = require('module/path');
```

如果您计划只编写标准的 ECMAScript `import`，这可能看起来并不是很重要，但在使用具有[条件导出](https://nodejs.org/api/packages.html#conditional-exports)的包时就会有所不同。

在 TypeScript 5.4 中，当将 `module` 设置为一个名为 `preserve` 的新选项时，可以使用 `require()`。

在 `--module preserve` 和 `--moduleResolution bundler` 之间，这两个选项更准确地模拟了像 Bun 等打包工具和运行时环境允许的操作以及它们如何执行模块查找。
实际上，在使用 `--module preserve` 时，`--moduleResolution` 选项将会隐式设置为 `bundler`（以及 `--esModuleInterop` 和 `--resolveJsonModule`）。

```json
{
  "compilerOptions": {
    "module": "preserve"
    // ^ also implies:
    // "moduleResolution": "bundler",
    // "esModuleInterop": true,
    // "resolveJsonModule": true,

    // ...
  }
}
```

在 `--module preserve` 下，ECMAScript 的 `import` 语句将始终按原样输出，而 `import ... = require(...)` 语句将被输出为 `require()` 调用（尽管实际上你可能不会使用 TypeScript 进行输出，因为你很可能会使用打包工具来处理你的代码）。
这一点不受包含文件的文件扩展名的影响。
因此，以下代码：

```ts
import * as foo from 'some-package/foo';
import bar = require('some-package/bar');
```

的输出结果会是这样：

```js
import * as foo from 'some-package/foo';
var bar = require('some-package/bar');
```

这也意味着您选择的语法将指定条件导出的匹配方式。
因此，在上面的示例中，如果 `some-package` 的 `package.json` 如下所示：

```json
{
  "name": "some-package",
  "version": "0.0.1",
  "exports": {
    "./foo": {
      "import": "./esm/foo-from-import.mjs",
      "require": "./cjs/foo-from-require.cjs"
    },
    "./bar": {
      "import": "./esm/bar-from-import.mjs",
      "require": "./cjs/bar-from-require.cjs"
    }
  }
}
```

TypeScript 会将路径解析为 `[...]/some-package/esm/foo-from-import.mjs` 和 `[...]/some-package/cjs/bar-from-require.cjs`。

更多详情请参考 [PR](https://github.com/microsoft/TypeScript/pull/56785)。

## 检查导入属性和断言

导入属性和断言现在会与全局的 `ImportAttributes` 类型进行检查。
这意味着运行时现在可以更准确地描述导入属性。

```ts
// In some global file.
interface ImportAttributes {
    type: "json";
}

// In some other module
import * as ns from "foo" with { type: "not-json" };
//                                     ~~~~~~~~~~
// error!
//
// Type '{ type: "not-json"; }' is not assignable to type 'ImportAttributes'.
//  Types of property 'type' are incompatible.
//    Type '"not-json"' is not assignable to type '"json"'.
```

感谢 [Oleksandr Tarasiuk](https://github.com/a-tarasyuk)的 [PR](https://github.com/microsoft/TypeScript/pull/56034)。

## 快速修复：添加缺失参数

TypeScript 现在提供了一个快速修复选项，可以为被调用时传递了过多参数的函数添加一个新的参数。

![](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2024/01/add-missing-params-5-4-beta-before.png)

![](https://devblogs.microsoft.com/typescript/wp-content/uploads/sites/11/2024/01/add-missing-params-5-4-beta-after.png)

当在多个现有函数之间传递一个新参数时，这将非常有用，而目前这样做可能会很麻烦。

感谢 [Oleksandr Tarasiuk](https://github.com/a-tarasyuk)的 [PR](https://github.com/microsoft/TypeScript/pull/56411)。

## 子路径导入支持自动导入

在 Node.js 中，`package.json` 通过一个名为 `imports` 的字段支持一种称为“子路径导入”的功能。
这是一种将包内的路径重新映射到其他模块路径的方式。
在概念上，这与路径映射非常相似，某些模块打包工具和加载器支持该功能（TypeScript 通过一个称为 `paths` 的功能也支持该功能）。
唯一的区别是，子路径导入必须始终以 `#` 开头。

TypeScript 的自动导入功能以前不会考虑 `imports` 中的路径，这可能令人沮丧。
相反，用户可能需要在 `tsconfig.json` 中手动定义路径。
然而，由于 [Emma Hamilton](https://github.com/emmatown) 的贡献，TypeScript 的自动导入现在支持[子路径导入](https://github.com/microsoft/TypeScript/pull/55015)！