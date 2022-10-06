---
origin: https://v8.dev/features/regexp-v-flag
date: 2022-10-06 13:21:48
---

## RegExp 的 `v` 标记，设置方式，以及字符串相关属性

JavaScript 从 ECMAScript3(1999 年) 开始支持正则表达式。16 年之后，ES2015 介绍了 [Unicode 模式（`u` 标志）][unicode mode]、[sticky 模式（`y` 标志）][sticky mode]，以及 [`RegExp.prototype.flags` getter][flags getter]。又过了三年，ES2018 介绍了
[`dotAll` 模式（`s` 标志）][dotall mode]、[后行断言][lookbehind assertions]、[具名捕获组][named capture groups]，以及 [Unicode 字符属性转义][unicode escapes]。在 ES2020，[`String.prototype.matchAll`][match all] 让正则表达式使用起来变得更加简单。JavaScript 正则表达式已经走了很长一段路，并仍然在不断改进中。

最新的例子演示了 [新的 `unicodeSets` 模式，用 `v` 标志开启][regexp v proposal]。这种新的模式解锁了对新的字符集的支持，包含如下功能：

- [字符串的 Unicode 属性](#字符串的-unicode-属性)
- [标记法 + 字符串字面量语法](#set-notation)
- [改进不区分大小写的匹配](#ignorecase)

这篇文章将深入研究以上问题，但最重要的是 --- 如何使用新的标记：

```ts
const re = /.../v;
```

`v` 标记可以和已存在的正则表达式的标记一起混用，但有一个明显的例外。`v` 标记启用了 `u` 标记中所有好的部分，但也带有额外的功能和改进 ---
其中一些和 `u` 标志不向后兼容。关键是，`v` 和 `u` 是完全独立的模式，而不是互补的关系。因此，`v` 和 `u` 不能同时使用 --- 尝试同时使用
两个标记在同一个正则表达式上会导致一个错误。仅有的有效选项是：用 `u` 或者用 `v`，或者既不用 `u` 也不用 `v`。但由于 `v` 是功能完整的选项，
所以很容易做这个选择...

让咱们深入研究新的功能吧！

## 字符串的 Unicode 属性

Unicode 标准为每个符号都分配了一个属性和属性值。例如，要获得用在希腊文字中的符号的集合，在 Unicode 数据库中搜索符号的 `Script_Extensions` 属性值中包含 `Greek` 即可。

ES2018 Unicode 字符属性转义使得在 ECMAScript 原生的正则表达式中访问 Unicode 字符属性成为可能。例如，表达式 `\p{Script_Extensions=Greek}` 匹配每一个用在希腊文字中的符号：

```ts
const regexGreekSymbol = /\p{Script_Extensions=Greek}/u;
regexGreekSymbol.test("π");
// → true
```

根据定义，Unicode 字符属性扩展到一组代码点，因此可以转换成一个字符类，其中每个字符类各自包含它们匹配的代码点。例如，`\p{ASCII_Hex_Digit}` 等效于 `[0-9a-fA-F]`：这每次仅能匹配一个
Unicode 字符/代码点，在某些条件下，这样并不高效。

```ts
// Unicode defines a character property named “Emoji”.
const re = /^\p{Emoji}$/u;

// Match an emoji that consists of just 1 code point:
re.test('⚽'); // '\u26BD'
// → true ✅

// Match an emoji that consists of multiple code points:
re.test('👨🏾‍⚕️'); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → false ❌
```

在上面的例子中，正则表达式不能匹配 👨🏾‍⚕️ 表情，因为它是由多个代码点组成的。而 `Emoji` 是一个 Unicode 字符属性。

幸运的是，Unicode 标准还定义了几个[字符属性][properties of strings]，这些属性扩展成一组字符串，其中每个字符串都包含一个或者多个代码点。

[unicode mode]: https://mathiasbynens.be/notes/es6-unicode-regex
[sticky mode]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/sticky#description
[dotall mode]: https://mathiasbynens.be/notes/es-regexp-proposals#dotAll
[flags getter]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/flags
[lookbehind assertions]: https://mathiasbynens.be/notes/es-regexp-proposals#lookbehinds
[named capture groups]: https://mathiasbynens.be/notes/es-regexp-proposals#named-capture-groups
[unicode escapes]: https://mathiasbynens.be/notes/es-unicode-property-escapes
[match all]: https://v8.dev/features/string-matchall
[regexp v proposal]: https://github.com/tc39/proposal-regexp-v-flag
[properties of strings]: https://www.unicode.org/reports/tr18/#domain_of_properties
