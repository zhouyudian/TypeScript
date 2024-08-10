# TypeScript 5.5

## 推断的类型谓词

TypeScript 的控制流分析在跟踪变量类型在代码中的变化时表现得非常出色：

```ts
interface Bird {
  commonName: string;
  scientificName: string;
  sing(): void;
}

// Maps country names -> national bird.
// Not all nations have official birds (looking at you, Canada!)
declare const nationalBirds: Map<string, Bird>;

function makeNationalBirdCall(country: string) {
  const bird = nationalBirds.get(country); // bird has a declared type of Bird | undefined
  if (bird) {
    bird.sing(); // bird has type Bird inside the if statement
  } else {
    // bird has type undefined here.
  }
}
```

通过让你处理 `undefined` 情况，TypeScript 促使你编写更健壮的代码。

在过去，这种类型细化在数组上更难应用。在所有以前的 TypeScript 版本中，这都会是一个错误：

```ts
function makeBirdCalls(countries: string[]) {
  // birds: (Bird | undefined)[]
  const birds = countries
    .map(country => nationalBirds.get(country))
    .filter(bird => bird !== undefined);

  for (const bird of birds) {
    bird.sing(); // error: 'bird' is possibly 'undefined'.
  }
}
```

代码是完全没有问题的：我们已经过滤掉了数组中所有的 `undefined` 值。
但是 TypeScript 却无法跟踪这些变化。

TypeScript 5.5 可以处理这种情况：

```ts
function makeBirdCalls(countries: string[]) {
  // birds: Bird[]
  const birds = countries
    .map(country => nationalBirds.get(country))
    .filter(bird => bird !== undefined);

  for (const bird of birds) {
    bird.sing(); // ok!
  }
}
```

注意 `birds` 变量的更精确类型。

因为 TypeScript 现在能够为 `filter` 函数推断出[类型谓词](https://www.typescriptlang.org/docs/handbook/2/narrowing.html#using-type-predicates)，所以这段代码才能工作。
你可以将代码提出到独立的函数中以便能清晰地看出这些：

```ts
// function isBirdReal(bird: Bird | undefined): bird is Bird
function isBirdReal(bird: Bird | undefined) {
  return bird !== undefined;
}
```

`bird is Bird` 是类型谓词。
它表示，如果函数返回 `true`，那么结果为 `Bird` (如果函数返回 `false`，结果为 `undefined`)。
`Array.prototype.filter` 的类型声明能够识别类型谓词，所以最终的结果是你获得了一个更精确的类型，并且代码通过了类型检查器的验证。

如果以下条件成立，TypeScript 会推断一个函数返回一个类型谓词：

- 函数没有显式的返回类型或类型谓词注解。
- 函数只有一个返回语句，并且没有隐式返回。
- 函数不会改变其参数。
- 函数返回的布尔表达式与参数的类型细化有关。

通常，这种推断方式会如你所预期的那样工作。以下是一些推断类型谓词的更多示例：

```ts
// const isNumber: (x: unknown) => x is number
const isNumber = (x: unknown) => typeof x === 'number';

// const isNonNullish: <T>(x: T) => x is NonNullable<T>
const isNonNullish = <T>(x: T) => x != null;
```

从前，TypeScript 仅会推断出这类函数返回 `boolean`。
但现在会推断出带类型谓词的签名，例如 `x is number` 或 `x is NonNullable<T>`。

类型谓词具有“当且仅当”的语义。
如果函数返回 `x is T`，那就意味着：

1. 如果函数返回 `true`，那么 `x` 的类型为 `T`。
1. 如果函数返回 `false`，那么 `x` 的类型不为 `T`。

如果你期待得到一个类型谓词但却没有，那么有可能违反了第二条规则。
这通常出现在“真值”检查中：

```ts
function getClassroomAverage(
  students: string[],
  allScores: Map<string, number>
) {
  const studentScores = students
    .map(student => allScores.get(student))
    .filter(score => !!score);

  return studentScores.reduce((a, b) => a + b) / studentScores.length;
  //     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  // error: Object is possibly 'undefined'.
}
```

TypeScript 没有为 `score => !!score` 推断出类型谓词，这是有道理的：如果返回 `true`，那么 `score` 是一个数字。但如果返回 `false`，那么 `score` 可能是 `undefined` 或者是数字（特别是 `0`）。这确实是一个漏洞：如果有学生在测试中得了零分，那么过滤掉他们的分数会导致平均分上升。这样一来，少数人会高于平均水平，而更多的人会感到沮丧！

与第一个例子一样，最好明确地过滤掉 `undefined` 值：

```ts
function getClassroomAverage(
  students: string[],
  allScores: Map<string, number>
) {
  const studentScores = students
    .map(student => allScores.get(student))
    .filter(score => score !== undefined);

  return studentScores.reduce((a, b) => a + b) / studentScores.length; // ok!
}
```

当对象类型不存在歧义时，“真实性”检查会为对象类型推断出类型谓词。
请记住，函数必须返回一个布尔值才能成为推断类型谓词的候选者：`x => !!x` 可能会推断出类型谓词，但 `x => x` 绝对不会。

显式类型谓词依然像以前一样工作。TypeScript 不会检查它是否会推断出相同的类型谓词。
显式类型谓词（"is"）并不比类型断言（"as"）更安全。

如果 TypeScript 现在推断出的类型比你期望的更精确，那么这个特性可能会破坏现有的代码。例如：

```ts
// Previously, nums: (number | null)[]
// Now, nums: number[]
const nums = [1, 2, 3, null, 5].filter(x => x !== null);

nums.push(null); // ok in TS 5.4, error in TS 5.5
```

解决方法是使用显式类型注解告诉 TypeScript 你想要的类型：

```ts
const nums: (number | null)[] = [1, 2, 3, null, 5].filter(x => x !== null);
nums.push(null); // ok in all versions
```

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57465) 和 [Dan 的博客](https://effectivetypescript.com/2024/04/16/inferring-a-type-predicate/)。

## 常量索引访问的控制流细化

当 `obj` 和 `key` 是常量时，TypeScript 现在能够细化 obj[key] 形式的表达式。

```ts
function f1(obj: Record<string, unknown>, key: string) {
  if (typeof obj[key] === 'string') {
    // Now okay, previously was error
    obj[key].toUpperCase();
  }
}
```

如上，`obj` 和 `key` 都没有修改过，因此 TypeScript 能够在 `typeof` 检查后将 `obj[key]` 细化为 `string`。
更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57847)。

## JSDoc `@import` 标签

如今，在 JavaScript 文件中，如果你只想为类型检查导入某些内容，这显得很繁琐。
JavaScript 开发者不能简单地导入一个名为 `SomeType` 的类型，如果在运行时该类型不存在的话。

```ts
// ./some-module.d.ts
export interface SomeType {
  // ...
}

// ./index.js
import { SomeType } from './some-module'; //  runtime error!

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

`SomeType` 类型在运行时不存在，因此导入会失败。
开发者可以使用命名空间导入来替代。

```ts
import * as someModule from './some-module';

/**
 * @param {someModule.SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

但 `./some-module` 仍是在运行时导入 - 可能不是期望的行为。

为避免此问题，开发者通常需要在 JSDoc 里使用 `import(...)`。

```ts
/**
 * @param {import("./some-module").SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

如果你想在多处重用该类型，你可以使用 `typedef` 来减少重覆。

```ts
/**
 * @typedef {import("./some-module").SomeType} SomeType
 */

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

对于本地使用 `SomeType` 的情况是没问题的，但是出现了很多重覆的导入并显得啰嗦。

因此，TypeScript 现在支持了新的 `@import` 注释标签，它与 ECMAScript 导入语句有相同的语法。

```ts
/** @import { SomeType } from "some-module" */

/**
 * @param {SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

此处，我们使用了命名导入。
我们也可将其写为命名空间导入。

```ts
/** @import * as someModule from "some-module" */

/**
 * @param {someModule.SomeType} myValue
 */
function doSomething(myValue) {
  // ...
}
```

因为它们是 JSDoc 注释，它们完全不影响运行时行为。

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/57207)。
感谢 [Oleksandr Tarasiuk](https://github.com/a-tarasyuk) 的贡献。

## 正则表达式语法检查

直到现在，TypeScript 通常会跳过代码中的大多数正则表达式。
这是因为正则表达式在技术上具有可扩展的语法，TypeScript 从未努力将正则表达式编译成早期版本的 JavaScript。
尽管如此，这意味着许多常见问题可能会在正则表达式中被忽略，并且它们要么会在运行时转变为错误，要么会悄悄地失败。

但是现在，TypeScript 对正则表达式进行基本的语法检查了！

```ts
let myRegex = /@robot(\s+(please|immediately)))? do some task/;
//                                            ~
// error!
// Unexpected ')'. Did you mean to escape it with backslash?
```

这是一个简单的例子，但这种检查可以捕捉到许多常见的错误。
事实上，TypeScript 的检查略微超出了语法检查。
例如，TypeScript 现在可以捕捉到不存在的后向引用周围的问题。

```ts
let myRegex = /@typedef \{import\((.+)\)\.([a-zA-Z_]+)\} \3/u;
//                                                        ~
// error!
// This backreference refers to a group that does not exist.
// There are only 2 capturing groups in this regular expression.
```

这同样适用于捕获命名的分组。

```ts
let myRegex =
  /@typedef \{import\((?<importPath>.+)\)\.(?<importedEntity>[a-zA-Z_]+)\} \k<namedImport>/;
//                                                                                        ~~~~~~~~~~~
// error!
// There is no capturing group named 'namedImport' in this regular expression.
```

TypeScript 现在还会检测到当使用某些 RegExp 功能时，这些功能是否比您的 ECMAScript 目标版本更新。
例如，如果我们在 ES5 目标中使用类似上文中的命名捕获组，将会导致错误。

```ts
let myRegex =
  /@typedef \{import\((?<importPath>.+)\)\.(?<importedEntity>[a-zA-Z_]+)\} \k<importedEntity>/;
//                                  ~~~~~~~~~~~~         ~~~~~~~~~~~~~~~~
// error!
// Named capturing groups are only available when targeting 'ES2018' or later.
```

同样适用于某些正则表达式标志。

请注意，TypeScript 的正则表达式支持仅限于正则表达式字面量。
如果您尝试使用字符串字面量调用 `new RegExp`，TypeScript 将不会检查提供的字符串。

更多详情请参考[PR](https://github.com/microsoft/TypeScript/pull/55600)。
感谢 [graphemecluster](https://github.com/graphemecluster/) 的贡献。
