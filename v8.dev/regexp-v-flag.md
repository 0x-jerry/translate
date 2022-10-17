---
origin: https://v8.dev/features/regexp-v-flag
date: 2022-10-06 13:21:48
---

## RegExp 的 `v` 标记，设置方式，以及字符串相关属性

JavaScript 从 ECMAScript3(1999 年) 开始支持正则表达式。16 年之后，ES2015 介绍了 [Unicode 模式（`u` 标志）][unicode mode]、[sticky 模式（`y` 标志）][sticky mode]，以及 [`RegExp.prototype.flags` getter][flags getter]。又过了三年，ES2018 介绍了
[`dotAll` 模式（`s` 标志）][dotall mode]、[后行断言][lookbehind assertions]、[具名捕获组][named capture groups]，以及 [Unicode 字符串属性转义][unicode escapes]。在 ES2020，[`String.prototype.matchAll`][match all] 让正则表达式使用起来变得更加简单。JavaScript 正则表达式已经走了很长一段路，并仍然在不断改进中。

最新的例子演示了 [新的 `unicodeSets` 模式，用 `v` 标志开启][regexp v proposal]。这种新的模式解锁了对新的字符集的支持，包含如下功能：

<!-- prevent markdown-all-in-one auto update toc -->

- ` `
- [字符串的 Unicode 属性](#字符串的-unicode-属性)
- [设置符号 + 字符串字面量语法](#设置符号--字符串字面量语法)
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

ES2018 Unicode 字符串属性转义使得在 ECMAScript 原生的正则表达式中访问 Unicode 字符串属性成为可能。例如，表达式 `\p{Script_Extensions=Greek}` 匹配每一个用在希腊文字中的符号：

```ts
const regexGreekSymbol = /\p{Script_Extensions=Greek}/u;
regexGreekSymbol.test("π");
// → true
```

根据定义，Unicode 字符串属性扩展到一组代码点，因此可以转换成一个字符类，其中每个字符类各自包含它们匹配的代码点。例如，`\p{ASCII_Hex_Digit}` 等效于 `[0-9a-fA-F]`：这每次仅能匹配一个
Unicode 字符/代码点，在某些条件下，这样并不高效。

```ts
// Unicode defines a character property named “Emoji”.
const re = /^\p{Emoji}$/u;

// Match an emoji that consists of just 1 code point:
re.test("⚽"); // '\u26BD'
// → true ✅

// Match an emoji that consists of multiple code points:
re.test("👨🏾‍⚕️"); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → false ❌
```

在上面的例子中，正则表达式不能匹配 👨🏾‍⚕️ 表情，因为它是由多个代码点组成的。而 `Emoji` 是一个 Unicode 字符串属性。

幸运的是，Unicode 标准还定义了一些[字符串属性][properties of strings]，这些属性扩展成一组字符串，其中每个字符串都包含一个或者多个代码点。 在正则表达式中，字符串属性翻译成一组字符集。为了说明这一点，想象一下，一个 Unicode 属性表示这些字符串，`'a'`, `'b'`, `'c'`, `'W'`, `'xy'`, 和 `'xyz'`。这些属性转换成以下正则表达式之一：`xyz|xy|a|b|c|W` 或者 `xyz|xy|[a-cW]`。
（最长的字符串优先，这样像 `xy` 这样的前缀就不会隐藏像 `xyz` 这样一个更长的字符串。）
不像已存在的 Unicode 属性转义，这个模式可以匹配多字符串。下面是一个使用字符串属性的例子：

```ts
const re = /^\p{RGI_Emoji}$/v;

// Match an emoji that consists of just 1 code point:
re.test("⚽"); // '\u26BD'
// → true ✅

// Match an emoji that consists of multiple code points:
re.test("👨🏾‍⚕️"); // '\u{1F468}\u{1F3FE}\u200D\u2695\uFE0F'
// → true ✅
```

这段代码片段引用了字符串属性 `RGI_Emoji`，Unicode 将其定义为「用于平时交流的所有有效的表情符号（字符序列）」。有了这个，我们现在可以
匹配表情符号，不管它包含了多少个代码点。

`v` 标记从一开始支持如下 Unicode 字符串属性：

- `Basic_Emoji`
- `Emoji_Keycap_Sequence`
- `RGI_Emoji_Modifier_Sequence`
- `RGI_Emoji_Flag_Sequence`
- `RGI_Emoji_Tag_Sequence`
- `RGI_Emoji_ZWJ_Sequence`
- `RGI_Emoji`

随着 Unicode 标准定义其他的字符串属性，这个受支持的列表将来可能会增加。尽管当前字符串属性都与表情符号相关，
将来的字符串属性可能用于完全不同的用例。

> 注意：尽管字符串属性目前受限于新的 `v` 标记，[但我们计划最终将使它在 `u` 模式也可用][proposal regexp v flag]。

## 设置符号 + 字符串字面量语法

当使用 `\p{...}` 转义（无论是字符属性还是新的字符串属性）它可以执行差/减或者交运算。

有了 `v` 标志，字符类就可以嵌套了，这些集合操作现在可以在字符类中执行，而不必使用相邻的前向断言或后向断言或冗长字符类来表示计算范围。

### 差/减 运算 `--`

语法 `A--B` 可以用于匹配 A 中的字符串，而不是 B 中的字符串。也就是说，差/减。

例如，如果你想要匹配除了 `π` 之外的所有的希腊符号，怎么做呢？用设置符号方法，解决这个问题就很简单：

```ts
/[\p{Script_Extensions=Greek}--π]/v.test("π"); // → false
```

使用 `--` 做差/减 运算，正则表达式引擎帮你做了复杂的工作，同时保持了代码的可读性和可维护性。

如果我们想要减去字符集 `α`, `β`, 和 `γ` 而不是单个字符呢？没问题 - 我们可以用嵌套的字符类并减去它的内容：

```ts
/[\p{Script_Extensions=Greek}--[αβγ]]/v.test("α"); // → false
/[\p{Script_Extensions=Greek}--[α-γ]]/v.test("β"); // → false
```

另一个例子是匹配非 ASCII 数字，例如稍后将它们转换成 ASCII 数字：

```ts
/[\p{Decimal_Number}--[0-9]]/v.test("𑜹"); // → true
/[\p{Decimal_Number}--[0-9]]/v.test("4"); // → false
```

设置符号也可以和新的字符串属性一起使用：

```ts
// 注意: 🏴󠁧󠁢󠁳󠁣󠁴󠁿 包含 7 个代码点。

/^\p{RGI_Emoji_Tag_Sequence}$/v.test("🏴󠁧󠁢󠁳󠁣󠁴󠁿"); // → true
/^[\p{RGI_Emoji_Tag_Sequence}--\q{🏴󠁧󠁢󠁳󠁣󠁴󠁿}]$/v.test("🏴󠁧󠁢󠁳󠁣󠁴󠁿"); // → false
```

这个例子匹配除了苏格兰的国旗之外的任何 RGI 表情符号标签序列。注意 `\q{...}` 的使用，这是字符串字面量在字符类里面的另外一种新的语法。
例如，`\q{a|bc|def}` 匹配字符串 `a`, `bc` 和 `def`。如果没有 `|q{...}`，它就不可能减去硬编码的多字符字符串

### 交集和 `&&`

`A&&B` 的语法匹配同时在 `A` 和 `B` 中的字符串，也就是交集。这让你可以做类似匹配希腊字母的事情：

```ts
const re = /[\p{Script_Extensions=Greek}&&\p{Letter}]/v;
// U+03C0 GREEK SMALL LETTER PI
re.test("π"); // → true
// U+1018A GREEK ZERO SIGN
re.test("𐆊"); // → false
```

匹配所有 ASCII 空格：

```ts
const re = [\p{White_Space}&&\p{ASCII}];
re.test('\n'); // → true
re.test('\u2028'); // → false
```

或者匹配所有蒙古数字：

```ts
const re = [\p{Script_Extensions=Mongolian}&&\p{Number}];
// U+1817 蒙古数字 7
re.test('᠗'); // → true
// U+1834 蒙古字母 CHA
re.test('ᠴ'); // → false
```

### 并集

匹配单字符字符串属于 `A` 或者属于 `B` 在之前已经可以用像 `[\p{Letter}\p{Number}]` 这样的字符类。有了 `v` 标志，
这个功能变得更加强大，因为现在它可以和字符串属性或者字符串字面量结合：

```ts
const re = /^[\p{Emoji_Keycap_Sequence}\p{ASCII}\q{🇧🇪|abc}xyz0-9]$/v;

re.test("4️⃣"); // → true
re.test("_"); // → true
re.test("🇧🇪"); // → true
re.test("abc"); // → true
re.test("x"); // → true
re.test("4"); // → true
```

该模式中的字符类组合为：

- 一个字符串属性（`\p{Emoji_Keycap_Sequence}`）
- 一个字符属性（`\p{ASCII}`）
- 多码点字符串 🇧🇪 和 abc 的字符串字面量语法
- 用于单个字符的 x, y 和 z 的经典字符类语法
- 用于字符范围的 0 到 9 的经典字符类语法

<!--  -->

另一个例子是匹配所有常用的表情符号标志，不管他们被编码未两个 ISO 代码字符（`RGI_Emoji_Flag_Sequence`）
还是被编码为一个特殊大小写的序列（`RGI_Emoji_Tag_Sequence`）：

```ts
const reFlag = /[\p{RGI_Emoji_Flag_Sequence}\p{RGI_Emoji_Tag_Sequence}]/v;
// A flag sequence, consisting of 2 code points (flag of Belgium):
re.test("🇧🇪"); // → true
// A tag sequence, consisting of 7 code points (flag of England):
re.test("🏴󠁧󠁢󠁥󠁮󠁧󠁿"); // → true
// A flag sequence, consisting of 2 code points (flag of Switzerland):
re.test("🇨🇭"); // → true
// A tag sequence, consisting of 7 code points (flag of Wales):
re.test("🏴󠁧󠁢󠁷󠁬󠁳󠁿"); // → true
```

### 改进不区分大小写的匹配

ES2015 `u` 标志存在[混淆的不区分大小写匹配行为](https://github.com/tc39/proposal-regexp-v-flag/issues/30)。思考以下两个正则表达式：

```ts
const re1 = /\p{Lowercase_Letter}/giu;
const re2 = /[^\p{lowercase_letter}]/giu;
```

第一个模式匹配所有小写字母。第二个模式使用 `\P` 代替 `\p` 来匹配所有除了小写字母以外的所有字符，
但是紧接着包在一个否定字符类中（`[^...]`）。通过设置 `i` 标志（不区分大小写），这两个正则表达式都变得不区分大小写。

直观地，你可能期望两个整个表达式有同样的行为。实际上，他们的行为非常不同：

```ts
const string = "aAbBcC4#";

string.replaceAll(re1, "X");
// → 'XXXXXX4#'

string.replaceAll(re2, "X");
// → 'aAbBcC4#''
```

新的 `v` 标志没有令人惊讶的行为。用 `v` 标志代替 `u` 标志。两种模式的行为一致：

```ts
const re1 = /\p{Lowercase_Letter}/giv;
const re2 = /[^\p{lowercase_letter}]/giv;

const string = "aAbBcC4#";

string.replaceAll(re1, "X");
// → 'XXXXXX4#'

string.replaceAll(re2, "X");
// → 'XXXXXX4#'
```

更普遍的说，无论是否设置 `i` 标志，`v` 标记都让 `[^\p{x}]` ≍ `[\P{X}]` ≍ `\P{X}` 而且让 `[^\p{x}]` ≍ `[\p{X}]` ≍ `\p{X}`。

# 扩展阅读

[提案仓库](https://github.com/tc39/proposal-regexp-v-flag)包含更多关于这些特性和设计决策的细节和背景。

作为我们在这些 JavaScript 功能上工作的一部分，我超越了“仅仅”提议对 ECMAScript 规范进行更改。我们将“字符串属性”的定义
升级到了 [Unicode UTS#18](https://unicode.org/reports/tr18/#Notation_for_Properties_of_Strings)，以便其他
编程语言可以统一实现类似的功能。我们还给 HTMl 标准提议进行修改，目的是在模式属性中也启用这些功能。

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
[proposal regexp v flag]: https://github.com/tc39/proposal-regexp-v-flag/issues/49
