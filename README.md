# BigInt: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 1 (?)

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Igor Sheludko, Benedikt Meurer, Michael Saboff, Adam Klein and others.

## Code samples:

Find the nth prime:

```js
// Takes a BigInt as an argument and returns a BigInt
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
  var cast = stdlib.BigInt.asUnsignedIntWithWidth;
  var values = new stdlib.Uint64Array(buffer);
  aIndex = aIndex|0;
  bIndex = bIndex|0;
  var aValue = cast(values[aIndex>>3], 64);
  var bValue = cast(values[bIndex>>3], 64);
  return cast(a + b, 64);
}
```

## Use cases

In many cases in JavaScript coding, integers larger than 2<sup>53</sup> come up, where casting to a double-precision float would lose real, relevant data:
- Cases which would fit in a signed or unsigned 64-bit integer
  - Reading certain machine registers, wire protocols
  - Protobufs or JSON documents that have GUIDs in them
  - Cryptography calculations
  - `stat` may give some data as 64-bit integers
  - Accurate timestamps
- Bigger than 64-bit int cases
  - Cryptography calculations implemented directly
  - Basis for implementing BigDecimal; perhaps some applications would be happy with fixpoint calculations based on a BigInt
  - Any mathematical calculation with larger integers, e.g., solving Project Euler problems

This proposal provides a new, second primitive numeric type, `BigInt`, to meet those use cases.

## API outline

BigInts are a new numerical type supporting literals (`1234n`) and arithmetic with operator overloading (`1n + 2n` makes `3n`).

### Literals, operators and their meanings

BigInts are, logically, arbitrary mathematic integers, with operator definitions which fall out naturally:
- Binary `+`, `-`, `*` and `**` find their mathematical answer when applied to two BigInts
- `/` and `%` round towards 0
- Bitwise operations `|`, `&`, `<<`, `>>`, `^` operate logically
  - Negative numbers to be interpreted as infinite-length two's complement.
- When applied to two BigInts, comparison operators `==`, `===`, `<`, `>`, `>=`, and `<=` perform a logical comparson
- Missing operators
  - `>>>` is not supported, as all BigInts are signed; to get an unsigned shift, pass in a positive BigInt to `>>`
  - `+` is unsupported on BigInts due to asm.js requirements

### The BigInt constructor

BigInts are a primitive type, and `BigInt` forms a constructor which can be used analogously to `Number`--to convert various types to BigInt values, as well as to be an object wrapper for BigInt values when used with property access.

When called as a function, it is similar to the `Number` constructor: It converts strings, Numbers, etc into BigInts.

#### Library functions

- `BigInt.asUnsignedIntWithWidth(BigInt, width)`: Wrap a BigInt between 0 and 2<sup>width</sup>-1
- `BigInt.asSignedIntWithWidth(BigInt, width)`: Wrap a BigInt between -2<sup>width-1</sup> and 2<sup>width-1</sup>-1
- `BigInt.parseInt(string[, radix])`: Analogous to `Number.parseInt`, to parse a BigNum from a String in any base.

### TypedArrays and DataViews

BigInts give JavaScript the ability to accurately represent 64-bit signed and unsigned integers:
- Uint64Array and Int64Array, whose elements read from property access are BigInts
- DataView.prototype.getInt64/getUint64, returning a BigInt

### No implicit conversions or mixed operands

A key design decision is to disallow mixed operations between BigInts and Numbers. The driving factor: Any implicit coercions would lose information.

When adding two values of different numeric types, between large integers and floating point numbers, the mathematical value of the result may be outside of the domain of either. For example, `(2n\*\*53n + 1n) + 0.5` has a value which cannot be accurately represented in either range. Floating point arithmetic is not the exact mathematical value, but at least it is well-defined by IEEE 754. The entire aim of introducing BigInt is, on the other hand, to make a way to preserve integer precision of larger values.

Many (all?) other dynamically typed programming languages which have multiple numeric types implement a *numeric tower*. This forms an ordering between types--on the built-in numeric types, when an operator is used with operands from two types, the greater type is chosen as the domain, and the "less general" operand is cast to the "more general" type. Unfortunately, as the previous example shows, there is no "more general" type between arbitrary integers and double-precision floats. The typical resolution, then, is to take floats as the "more general" type.

Silently losing precision sometimes may be a problem, but in most dynamically typed programming languages which provide integers and floats, integers are written like `1` and floats are written like `1.0`. It's possible to scan code for operations which may introduce floating point precision by looking for a decimal point. JavaScript exacerbates the scope of losing precision by making the unfortunate decision that a simple literal like `1` is a float. So, if mixed-precision were allowed, an innocent calculation such as `2n ** 53n + 1` would produce the float `2**53`--defeating the core functionality of this feature.

To avoid this problem, this proposal bans implicit coercions between Numbers and BigInts, including operations which are mixed type. `1n + 1` throws a TypeError. So does passing `1n` as an argument into any JavaScript standard library function or Web API which expects a Number. Instead, to convert between types, an explicit call to `Number()` or `BigInt()` needs to be made.

## Design goals

### Don't break user intuition

{concrete risks for user intuition}

### Don't break math

Precision, so no mixed operands

### Don't break asm.js

+ needs to always return Numbers, >>> 0 needs to always return Integers in the range

### Don't break potential future value types extensions

- Should pave the cowpath with value types, in conjunction with the work done on SIMD.js.
-- Primitives and wrappers as in SIMD, other value types proposals
-- Operator overloading without any use of built-in double dispatch 
- Leave L open as a suffix for an Int64 type

### Don't break JavaScript ergonomics

This proposal comes with built-in operator overloading

### Don't break a consistent model of JavaScript

New primitive type with wrappers, just like how Symbol was added

### Don't break the web

We need to choose a web-compatible name to add to the global object; this means that Integer might be unacceptable.

## Open questions

### Mixed comparison operators

It would be mathematically well-defined to allow comparison operators such as `<` compare between Numbers and 

### Semantics of bitwise operations

Here, we use 

### Bikeshedding

The literal suffix n is reserved...

I'm calling it BigInt here; Integer may be web-incompatble; BigNum sounds too much like Number.

## Design alternatives

### Int64/Uint64

Previously, signed and unsigned integer types ...

### Allowing mixed operands

(redundant with an above section?)

### Leave out TypedArrays and DataView methods for now

One possibility would be to wait until a potential future Int64/Uint64 proposal is created for TypedArray classes and DataView methods. However, they are included here, following the pattern of existing TypedArrays which return Numbers--a more general type than the contents, but one which accurately represents it.

## Left for future proposals

### Function and constant library

It would be reasonable to add integer-related mathematical library functions, especially those which could be more efficiently based on instructions found on CPUs likely to be used by implementations. This includes:
- Find first set/unset bit
- Popcount
- Find most significant set/unset bit
- Convenience functions for doing arithmetic in a specific modulus (e.g., 64-bit signed or unsigned) rather than 
- Constants for the maximum and minimum 64-bit signed and unsigned integer 
- A library function 

### Any other numerical types, and generalization to binary data and value types

In the course of development of ES2015, the proposal to add 

## Implementation status

V8 is interested in implementing BigInt as soon as the major semantics are worked out.
