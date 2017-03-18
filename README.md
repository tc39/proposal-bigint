# Integer: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 1 (?)

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Jaro Sevcik, Benedikt Meurer, Michael Saboff, Adam Klein and others.

## Code samples:

Find the nth prime:

```js
// Takes an Integer as an argument and returns an Integer
function nthPrime(nth) {
  function isPrime(p) {
    for (let i = 2n; i < p; i++) {
      if (p % i === 0n) return false;
    }
    return true;
  }
  for (let i = 2n; ; i++) {
    if (isPrime(i)) {
      if (--nth === 0n) return i;
    }
  }
}
```

Read and add two unsigned 64-bit integers in asm.js on the heap:

```js
function add64(stdlib, buffer, aIndex, bIndex) {
  "use asm";
  var cast = stdlib.Integer.asUintN;
  var values = new stdlib.Uint64Array(buffer);
  aIndex = aIndex|0;
  bIndex = bIndex|0;
  var aValue = values[aIndex>>3];
  var bValue = values[bIndex>>3];
  return cast(64, aValue + bValue);
}
```

## Use cases

In many cases in JavaScript coding, integers larger than 2<sup>53</sup> come up, where casting to a double-precision float would lose real, relevant data:
- Cases which would fit in a signed or unsigned 64-bit integer
  - Reading certain machine registers, wire protocols
  - Protobufs or JSON documents that have GUIDs in them
  - Highly optimized cryptography calculations
  - `stat` may give some data as 64-bit integers
  - Accurate timestamps
- Bigger than 64-bit int cases
  - Cryptography calculations implemented directly
  - Basis for implementing BigDecimal; perhaps some applications would be happy with fixpoint calculations based on an Integer
  - Any mathematical calculation with larger integers, e.g., solving Project Euler problems

This proposal provides a new, second primitive numeric type, `Integer`, to meet those use cases.

## API outline

Integer is a new numerical type supporting literals (`1234n`) and arithmetic with operator overloading (`1n + 2n` makes `3n`).

### Literals, operators and their meanings

Integers are, logically, arbitrary mathematic integers, with operator definitions which fall out naturally:
- Binary `+`, `-`, `*` and `**` find their mathematical answer when applied to two Integers
- `/` and `%` round towards 0
- Bitwise operations `|`, `&`, `<<`, `>>`, `^` operate logically
  - Negative numbers to be interpreted as infinite-length two's complement (see [discussion](https://github.com/littledan/proposal-integer/issues/3)).
- When applied to two Integers, comparison operators `==`, `===`, `<`, `>`, `>=`, and `<=` perform a mathematical comparson
- Missing operators
  - `>>>` is not supported, as all Integers are signed; to get an unsigned shift, pass in a positive Integer to `>>` ([discussion](https://github.com/littledan/proposal-integer/issues/6))
  - Unary `+` is unsupported on Integers due to asm.js requirements; details explained below
- In a conditional, `if (0n)` executes the `else` branch.

Literals for Integers are similar to Number literals, but followed by `n`. They can be written with binary, octal or hexadecimal notation, e.g., `0x100n`. Legacy octal syntax (`0640`) is not allowed, only new-style (`0o064n`).

### The Integer constructor

Integer is a primitive type, and `Integer` forms a constructor which can be used analogously to `Number`--to convert various types to Integer values, as well as to be an object wrapper for Integer values when used with property access.

When called as a function, it is similar to the `Number` constructor: It converts strings, Numbers, etc into Integers.

#### Library functions

- `Integer.asUintN(width, Integer)`: Wrap an Integer between 0 and 2<sup>width</sup>-1
- `Integer.asIntN(width, Integer)`: Wrap an Integer between -2<sup>width-1</sup> and 2<sup>width-1</sup>-1
- `Integer.parseInt(string[, radix])`: Analogous to `Number.parseInt`, to parse an Integer from a String in any base.

### TypedArrays and DataViews

Integers give JavaScript the ability to accurately represent 64-bit signed and unsigned integers:
- Uint64Array and Int64Array, whose elements read from property access are Integers
- DataView.prototype.getInt64/getUint64, returning an Integer

Similarly, Integers may be used by the WebAssembly FFI for 64-bit arguments and return values to functions.

### No implicit conversions or mixed operands

A key design decision is to disallow mixed operations between Integers and Numbers. The driving factor: Any implicit coercions would lose information.

When adding two values of different numeric types, between large integers and floating point numbers, the mathematical value of the result may be outside of the domain of either. For example, `(2n**53n + 1n) + 0.5` has a value which cannot be accurately represented in either range. Floating point arithmetic is not the exact mathematical value, but at least it is well-defined by IEEE 754. The entire aim of introducing Integer is, on the other hand, to make a way to preserve integer precision of larger values.

Many (all?) other dynamically typed programming languages which have multiple numeric types implement a *numeric tower*. This forms an ordering between types--on the built-in numeric types, when an operator is used with operands from two types, the greater type is chosen as the domain, and the "less general" operand is cast to the "more general" type. Unfortunately, as the previous example shows, there is no "more general" type between arbitrary integers and double-precision floats. The typical resolution, then, is to take floats as the "more general" type.

Silently losing precision sometimes may be a problem, but in most dynamically typed programming languages which provide integers and floats, integers are written like `1` and floats are written like `1.0`. It's possible to scan code for operations which may introduce floating point precision by looking for a decimal point. JavaScript exacerbates the scope of losing precision by making the unfortunate decision that a simple literal like `1` is a float. So, if mixed-precision were allowed, an innocent calculation such as `2n ** 53n + 1` would produce the float `2**53`--defeating the core functionality of this feature.

To avoid this problem, this proposal bans implicit coercions between Numbers and Integers, including operations which are mixed type. `1n + 1` throws a TypeError. So does passing `1n` as an argument into any JavaScript standard library function or Web API which expects a Number. Instead, to convert between types, an explicit call to `Number()` or `Integer()` needs to be made to decide which domain to operate in. `0 === 0n` returns `false`, and `0 == 0n` and `0 < 0n` throw a `TypeError`.

## Design goals

### Don't break user intuition

When a messy situation comes up, the this proposal errs on the side of throwing an exception rather than silently giving a bad answer. This is what's behind throwing a TypeError on adding an Integer and a Number: If we don't have a good answer, better to not give one.

### Don't break math

The semantics of all operators should ideally be based on some mathematical first principles, and certainly be well-defined, rather than excessively exposing implementation artifacts. `/` and `%` round towards 0, this is to match well-established computer conventions; aside from that, all operators have clean, mathematical definitions which don't appeal to the implementation shape.

### Don't break asm.js

Although this proposal introduces operator overloading, it throws in any of the cases that asm.js depends on for setting up type checking. asm.js relies on a few identies:
- Unary `+` followed by an expression is always either a Number, or results in throwing. For this reason, unfortunately, `+` on an Integer needs to throw, rather than being symmetrical with `+` on Number: Otherwise, previosly "type-declared" asm.js code would now be polymorphic.
- `|0` always returns a Number in int32 range, or throws. This proposal maintains that, as it would throw on an Integer for being a mixed operand type.
- `>>> 0` always returns a Number in uint32 range, throwing as `>>>` is not supported on Integer at all.
- `Math.fround` always returns a Number in float32 range, or throws. This proposal would throw if `Math.fround` is called with an Integer, preserving the property.

This proposal makes special allowances to make Integers usable in asm.js code to build support for 64-bit integers, by including the standard library functions `Integer.asUintN` and `Integer.asIntN` as well as `Uint64Array` and `Int64Array`.

### Don't break potential future value types extensions

- Should pave the cowpath to value types, as previously discussed, in conjunction with the work done on SIMD.js.
 - Integers are a new primitive type, and have associated wrappers, as do the other primitives, and SIMD.js, and as value types would get.
 - Operator overloading on value types may follow a similar pattern of requiring uniform argument types; this avoids the very difficult proposition of double dispatch. By not supporting mixed operands, Integer gets no superpowers which would be very difficult to generalize.
- `L` has been proposed as a literal suffix for positive Int64 values. This proposal uses `n` to leave that space free for later (bikeshedding welcome!).

### Don't break JavaScript ergonomics

This proposal comes with built-in operator overloading in order to not make Integers too ugly to be usable. One particular hazard, if Integers were to be operated on with static methods, is that users may convert the Integer into a Number in order to use the `+` operator on it--this would work most of the time, just not with big enough values, so it might pass tests. By including operator overloading, it would be even shorter code to add the Integers properly than to convert them to Numbers, which minimizes the chance of this bug.

### Don't break a consistent model of JavaScript

This proposal adds a new primitive type with wrappers, similar to Symbol. As part of integrating Integers into the JavaScript specification, a high amount of rigor will be required to differentiate three types floating around in the specification: Mathematical values, Integers and Numbers.

### Don't break the web

We need to choose a web-compatible name to add to the global object; this means that `Integer` is likely to be an unacceptable name. Otherwise, this proposal doesn't seem to have much compatibility risk; `0n` was previously a syntax error.

### Don't break good performance

Design work here is being done in conjunction with planned prototyping in V8; this will be used to develop feedback to ensure that the proposal is efficiently implementable.

## Open questions

### Mixed comparison operators ([bug](https://github.com/littledan/proposal-integer/issues/2))

It would be mathematically well-defined to allow comparison operators such as `<` and `==` compare between Numbers and Integers. Unlike operators like `+`, there is no loss of precision, since the output is just a Boolean.

A couple arguments for why we should throw on `0 < 0n` and `0 == 0n`:
- It's more straightforward to imagine how single-dispatch, matching-only operator overloading could be generalized to user-defined types and objects, whereas multiple dispatch is more difficult.
- Throwing here leads to a simple, regular rule: You never have any operator at all work between types, just as it is prohibited for `+`. The simplicity of the rule may help developers understand what they can do.
- Not supporting it is a "more minimal" proposal; we can likely to back and support mixed operands here, but it would be hard to go in the other direction.

With "MVP" as a tiebreaker, this proposal starts with banning mixed operations.

### Bikeshedding ([bug](https://github.com/littledan/proposal-integer/issues/1))

This proposal uses `n` as the literal suffix; `N`, `I`, `L` and `l` are some other proposed alternatives.

This proposal calls the class `Integer`; other proposed alternatives are `BigInt` and `BigNum`. An earlier version was called `BigInt`.

### And much, much more

There are many [open bugs](https://github.com/littledan/proposal-integer/issues) to discuss various issues. Come and join the discussion, and file some of your own!

## Design alternatives not proposed here

### Int64/Uint64

Brendan Eich previously proposed two types--signed and unsigned Int64/Uint64, for JavaScript. These meet many of the concrete use cases for Integers. One claim is that they may provide more predictable performance; however, my understanding is that, if the appropriate casting operator (e.g., `Integer.asUintN`) is used everywhere, an implementation like V8 is expected provide the same performance for Integer as it would for an Int64 type. The risks of this approach are that the performance won't pan out without harder-to-remove cliffs, or that the ergonomics will be too bad for performance-sensitive code--we'll watch for these risks as the prototype advances.

### Allowing mixed operands

We could allow mixed operands like `1 + 1n` returning `2` (a Number). It would lose precision, be harder to implement with as high performance, and not generalize well to user-defined types, but it would follow the well-worn path of many other programming languages, which have just let users deal with these issues.

### Leave out TypedArrays and DataView methods for now

One possibility would be to wait until a potential future Int64/Uint64 proposal is created for TypedArray classes and DataView methods. However, they are included here, following the pattern of existing TypedArrays which return Numbers--a more general type than the contents, but one which accurately represents it.

## Left for future proposals

### Function and constant library ([bug](https://github.com/littledan/proposal-integer/issues/20))

It would be reasonable to add integer-related mathematical library functions, especially those which could be more efficiently based on instructions found on CPUs likely to be used by implementations. This includes:
- Find first set/unset bit
- Popcount
- Find most significant set/unset bit
- Convenience functions for doing arithmetic in a specific modulus (e.g., 64-bit signed or unsigned) rather than requiring use of the wrap functions and arithmetic separately.
- Constants for the maximum and minimum 64-bit signed and unsigned integer

### Any other numerical types, and generalization to binary data and value types

In the course of development of ES2015, the proposal to add 64-bit integers was generalized significantly into a value types/binary data proposal. This big proposal became very complicated and unwieldy, so it was dropped from ES2015. The current proposal is much smaller, and could be built on incrementally. Value types and smaller integer types may be possible follow-ons, and this proposal is designed to generalize to those, but not block on them.

## Implementation status

No current (known) implementations exist.

V8 is interested in implementing Integer as soon as the major semantics are worked out.

## Specification

See the [specification](https://littledan.github.io/proposal-integer/) for more fine details.
