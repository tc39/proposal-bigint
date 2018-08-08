# BigInt: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 3

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Jaro Sevcik, Benedikt Meurer, Michael Saboff, Adam Klein and others.

## API outline

BigInts are a new numerical type supporting literals (`1234n`) and arithmetic with operator overloading (`1n + 2n` makes `3n`).

### Literals, operators and their meanings

BigInts are, logically, arbitrary mathematic integers, with operator definitions which fall out naturally:
- Binary `+`, `-`, `*` and `**` find their mathematical answer when applied to two BigInts
- `/` and `%` round towards 0
- Bitwise operations `|`, `&`, `<<`, `>>`, `^` operate logically
  - Negative numbers to be interpreted as infinite-length two's complement (see [discussion](https://github.com/tc39/proposal-bigint/issues/3)).
- When applied to two BigInts, comparison operators `==`, `===`, `<`, `>`, `>=`, and `<=` perform a mathematical comparison
- Missing operators
  - `>>>` is not supported, as all BigInts are signed; to get an unsigned shift, pass in a positive BigInt to `>>` ([discussion](https://github.com/tc39/proposal-bigint/issues/6))
  - Unary `+` is unsupported on BigInts due to asm.js requirements; details explained below
- In a conditional, `if (0n)` executes the `else` branch.

Literals for BigInts are similar to Number literals, but followed by `n`. They can be written with binary, octal or hexadecimal notation, e.g., `0x100n`. Legacy octal syntax (`0640`) is not allowed, only new-style (`0o064n`).

The choice of `BigInt` comes from an attempt to preserve user intuition about the lack of compatibility of `BigInt` with things that are currently based on `Number`s. Because implicit coercions lead to TypeErrors, the guidance is that existing users of Numbers should stay with Numbers if it works for the application, and only large usages need to upgrade to `BigInt`. The name `BigInt` is hoped to emphasize this not-by-default usage. The suffix `n` is basically arbitrary.

### The BigInt constructor

BigInts are a primitive type, and `BigInt` forms a constructor which can be used analogously to `Number`--to convert various types to BigInt values, as well as to be an object wrapper for BigInt values when used with property access.

When called as a function, it is similar to the `Number` constructor: It converts strings, Numbers, etc into BigInts.

#### Library functions

- `BigInt.asUintN(width, BigInt)`: Wrap a BigInt between 0 and 2<sup>width</sup>-1
- `BigInt.asIntN(width, BigInt)`: Wrap a BigInt between -2<sup>width-1</sup> and 2<sup>width-1</sup>-1

### TypedArrays and DataViews

BigInts give JavaScript the ability to accurately represent 64-bit signed and unsigned integers:
- BigUint64Array and BigInt64Array, whose elements read from property access are BigInts
- DataView.prototype.getBigInt64/getBigUint64, returning a BigInt

Similarly, BigInts may be used by the WebAssembly FFI for 64-bit arguments and return values to functions.

### No implicit conversions or mixed operands

A key design decision is to disallow mixed operations between BigInts and Numbers. The driving factor: Any implicit coercions would lose information.

When adding two values of different numeric types, between large integers and floating point numbers, the mathematical value of the result may be outside of the domain of either. For example, `(2n**53n + 1n) + 0.5` has a value which cannot be accurately represented in either range. Floating point arithmetic is not the exact mathematical value, but at least it is well-defined by IEEE 754. The entire aim of introducing BigInt is, on the other hand, to make a way to preserve integer precision of larger values.

Many (all?) other dynamically typed programming languages which have multiple numeric types implement a *numeric tower*. This forms an ordering between types--on the built-in numeric types, when an operator is used with operands from two types, the greater type is chosen as the domain, and the "less general" operand is cast to the "more general" type. Unfortunately, as the previous example shows, there is no "more general" type between arbitrary integers and double-precision floats. The typical resolution, then, is to take floats as the "more general" type.

Silently losing precision sometimes may be a problem, but in most dynamically typed programming languages which provide integers and floats, integers are written like `1` and floats are written like `1.0`. It's possible to scan code for operations which may introduce floating point precision by looking for a decimal point. JavaScript exacerbates the scope of losing precision by making the unfortunate decision that a simple literal like `1` is a float. So, if mixed-precision were allowed, an innocent calculation such as `2n ** 53n + 1` would produce the float `2**53`--defeating the core functionality of this feature.

To avoid this problem, this proposal bans implicit coercions between Numbers and BigInts, including operations which are mixed type. `1n + 1` throws a TypeError. So does passing `1n` as an argument into any JavaScript standard library function or Web API which expects a Number. Instead, to convert between types, an explicit call to `Number()` or `BigInt()` needs to be made to decide which domain to operate in. `0 === 0n` returns `false`.

Comparisons form an exception to this rule: It's mathematically well-defined to allow comparison operators such as `<` and `==` compare between Numbers and BigInts. Unlike operators like `+`, there is no loss of precision, since the output is just a Boolean. Further, `==` is extended in comparisons with strings in analogous ways, such as `0n == ""` is `true`. Although we may not be able to extend this support to user-defined types, it seems sufficiently important for meeting user expectations that this proposal adds them as a one-off.

## Design goals

#### Find a balance between maintaining user intuition and preserving precision

When a messy situation comes up, this proposal errs on the side of throwing an exception rather than silently giving a bad answer. This is what's behind throwing a TypeError on adding a BigInt and a Number: If we don't have a good answer, better to not give one.

Some JavaScript users will surely have the intuition that everything will just work, and that exceptions will not be thrown, when doing this sort of interoperation, even at the cost of losing precision. Axel Rauschmeyer [elaborated](https://gist.github.com/rauschma/13d48d1c49615ce2396ce7c9e45d4cd1) a proposal which elaborates this approach. [Further discussion](https://github.com/tc39/proposal-integer/issues/36) raised a few issues with this approach which are a direct result of upgrading the behavior of existing Numbers, both in terms of compatibility and reasonable expectations of invariants of JavaScript values. Given the tradeoffs, we have chosen to stick with the distinct type approach.

### Don't break math

The semantics of all operators should ideally be based on some mathematical first principles, and certainly be well-defined, rather than excessively exposing implementation artifacts. `/` and `%` round towards 0, this is to match well-established computer conventions; aside from that, all operators have clean, mathematical definitions which don't appeal to the implementation shape.

### Don't break asm.js

Although this proposal introduces operator overloading, it throws in any of the cases that asm.js depends on for setting up type checking. asm.js relies on a few identities:
- Unary `+` followed by an expression is always either a Number, or results in throwing. For this reason, unfortunately, `+` on a BigInt needs to throw, rather than being symmetrical with `+` on Number: Otherwise, previously "type-declared" asm.js code would now be polymorphic.
- `|0` always returns a Number in int32 range, or throws. This proposal maintains that, as it would throw on a BigInt for being a mixed operand type.
- `Math.fround` always returns a Number in float32 range, or throws. This proposal would throw if `Math.fround` is called with a BigInt, preserving the property.

Analogously, `>>> 0` always returns a Number in uint32 range, throwing as `>>>` is not supported on BigInt at all. Note: asm.js itself does not require this property, as `>>>` may be an overloaded operator, and `|0` is used for all `int` parameter declarations, but `>>> 0` is a common idiom to achieve this property in JavaScript code.

This proposal makes special allowances to make BigInt usable in asm.js code to build support for 64-bit integers, by including the standard library functions `BigInt.asUintN` and `BigInt.asIntN` as well as `BigUint64Array` and `BigInt64Array`.
 The operator overloading in this proposal should not complicate the asm.js model: asm.js already treats operators as "overloaded" between floats, doubles, and signed and unsigned integers.

### Don't break potential future value types extensions

- Should pave the cowpath to value types, as previously discussed, in conjunction with the work done on SIMD.js.
 - BigInts are a new primitive type, and have associated wrappers, as do the other primitives, and SIMD.js, and as value types would get.
 - Operator overloading on value types may follow a similar pattern of requiring uniform argument types; this avoids the very difficult proposition of double dispatch. By not supporting mixed operands, BigInt gets no superpowers which would be very difficult to generalize. Mixed comparisons are a one-off exception to this principle, however.
- `L` has been proposed as a literal suffix for positive Int64 values. This proposal uses `n` to leave that space free for later (bikeshedding welcome!).

### Don't break JavaScript ergonomics

This proposal comes with built-in operator overloading in order to not make BigInts too ugly to be usable. One particular hazard, if BigInts were to be operated on with static methods, is that users may convert the BigInt into a Number in order to use the `+` operator on it--this would work most of the time, just not with big enough values, so it might pass tests. By including operator overloading, it would be even shorter code to add the BigInts properly than to convert them to Numbers, which minimizes the chance of this bug.

### Don't break a consistent model of JavaScript

This proposal adds a new primitive type with wrappers, similar to Symbol. As part of integrating BigInts into the JavaScript specification, a high amount of rigor will be required to differentiate three types floating around in the specification: Mathematical values, BigInts and Numbers.

### Don't break the web

We need to choose a web-compatible name to add to the global object. There is some worry that the name `Integer` could have a web compatibility risk, though we don't actually have data about this. Changing the semantics of existing operations performed on existing numbers might also have compatibility risk, and this proposal avoids making these kinds of changes.

### Don't break good performance

Design work here is being done in conjunction with planned prototyping in V8; this will be used to develop feedback to ensure that the proposal is efficiently implementable.

## Design alternatives not selected here

### Int64/Uint64

Brendan Eich previously proposed two types--signed and unsigned Int64/Uint64, for JavaScript. These meet many of the concrete use cases for BigInts. One claim is that they may provide more predictable performance; however, my understanding is that, if the appropriate casting operator (e.g., `BigInt.asUintN`) is used everywhere, an implementation like V8 is expected provide the same performance for BigInt as it would for an Int64 type. The risks of this approach are that the performance won't pan out without harder-to-remove cliffs, or that the ergonomics will be too bad for performance-sensitive code--we'll watch for these risks as the prototype advances.

### Allowing mixed operands

We could allow mixed operands like `1 + 1n` returning `2` (a Number). It would lose precision, be harder to implement with as high performance, and not generalize well to user-defined types, but it would follow the well-worn path of many other programming languages, which have just let users deal with these issues.

### Leave out TypedArrays and DataView methods for now

One possibility would be to wait until a potential future Int64/Uint64 proposal is created for TypedArray classes and DataView methods. However, they are included here, following the pattern of existing TypedArrays which return Numbers--a more general type than the contents, but one which accurately represents it.

## Left for future proposals

### Function and constant library ([bug](https://github.com/tc39/proposal-bigint/issues/20))

It would be reasonable to add integer-related mathematical library functions, especially those which could be more efficiently based on instructions found on CPUs likely to be used by implementations. This includes:
- Bitcast between BigInt and Number
- Find first set/unset bit
- Popcount
- Find most significant set/unset bit
- Convenience functions for doing arithmetic in a specific modulus (e.g., 64-bit signed or unsigned) rather than requiring use of the wrap functions and arithmetic separately.
- Constants for the maximum and minimum 64-bit signed and unsigned integer

### Any other numerical types, and generalization to binary data and value types

In the course of development of ES2015, the proposal to add 64-bit integers was generalized significantly into a value types/binary data proposal. This big proposal became very complicated and unwieldy, so it was dropped from ES2015. The current proposal is much smaller, and could be built on incrementally. Value types and smaller integer types may be possible follow-ons, and this proposal is designed to generalize to those, but not block on them.

## Implementation status

- [V8](https://bugs.chromium.org/p/v8/issues/detail?id=6791) by Georg Neis and Jakob Kummerow
- [JSC](https://bugs.webkit.org/show_bug.cgi?id=175359) by Caio Lima and Robin Morisset
- [SpiderMonkey](https://bugzilla.mozilla.org/show_bug.cgi?id=1366287) by Robin Templeton

## Specification

See the [specification](https://tc39.github.io/proposal-bigint/) for more fine details.

### Related specification proposals

- [BigInt WebAssembly JS API integration proposal](https://github.com/WebAssembly/spec/pull/707)
- [HTML serialization of BigInt](https://github.com/whatwg/html/pull/3480)
- [BigInt as an IndexedDB key](https://github.com/w3c/IndexedDB/pull/231)
