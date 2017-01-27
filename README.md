# BigInt: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 1 (?)

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Igor Sheludko, Benedikt Meurer, Michael Saboff, Adam Klein and others.

## Problem statement

In many cases in JavaScript coding, integers larger than 2<sup>53</sup> come up, where casting to a double-precision float would lose real, relevant data:
- 64-bit int cases
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

### No implicit conversions or mixed operands

A key design decision is to disallow mixed operations between BigInts and Numbers. The driving factor: Any implicit coercions would lose information. JavaScript makes the unfortunate decision that a simple literal like `1` is a float. When adding a ...

### The BigInt constructor

BigInts are a primitive type, and `BigInt` forms a constructor which can be used analogously to `Number`--to convert various types to BigInt values, as well as to be an object wrapper for BigInt values when used with property access.

BigInt.asUnsignedIntWithWidth(BigInt, width)
BigInt.asSignedIntWithWidth(BigInt, width)
BigInt.parseInt(string[, radix])

### TypedArrays and DataViews

Int64 and Uint64 TypedArrays, DataView methods, returning BigInt

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
