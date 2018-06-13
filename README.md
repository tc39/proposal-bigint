# BigInt: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 3

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Jaro Sevcik, Benedikt Meurer, Michael Saboff, Adam Klein and others.

## Contents
1. What Is It?
2. How Does It Work?
	- Syntax
	- Example: Calculating Primes
	- Operators
	- Comparisons
	- Conditionals
	- Other API Notes
3. Gotchas & Exceptions
	- Interoperation with `Number` and `String`
	- Rounding
	- Other Exceptions
4. About the Proposal
- Motivation, Or Why Do We Need Such Big Numbers?
- Design Philosophy, Or Why Is This Like This?
- State of the Proposal


## What Is It?

`BigInt` is a new primitive that provides a way to represent whole numbers larger than 2^53, which is the largest number Javascript can reliably represent with the `Number` primitive.

```js
const x = Number.MAX_SAFE_INTEGER;
// ↪ 9007199254740991, this is 1 less than 2^53

const y = x + 1;
// ↪ 9007199254740992, ok, checks out

const z = x + 2
// ↪ 9007199254740992, wait, that’s the same as above!
```

[Learn more about how numbers are represented in Javascript.]() 

## How Does It Work?

### Syntax

A `BigInt` is created by appending `n` to the end of the integer or by calling the constructor.

```js

const theBiggestInt = 9007199254740991n;

const alsoHuge = BigInt(9007199254740991);
// ↪ 9007199254740991n

const hugeButString = BigInt('9007199254740991');
// ↪ 9007199254740991n

```

### Example: Calculating Primes

```js
// I think we can add a slightly simpler version of the primes example here
```

### Operators

You can use `+`, `*`, `-`, `**` and `%` with `BigInt`s, just like with `Number`s.

```js

const previousMaxSafe = BigInt(Number.MAX_SAFE_INTEGER);
// ↪ 9007199254740991

const maxPlusOne = previousMaxSafe + 1n;
// ↪ 9007199254740992n
 
const theFuture = previousMaxSafe + 2n;
// ↪ 9007199254740993n, this works now!

const multi = previousMaxSafe * 2n;
// ↪ 18014398509481982n

const subtr = multi – 10n;
// ↪ 18014398509481972n

const mod = multi % 10n;
// ↪ 2n

const bigN = 2n ** 54n;
// ↪ 18014398509481984n

bigN * -1n
// ↪ –18014398509481984n

```

The `/` operator also work as expected with whole numbers. However, since these are `BigInt`s and not `BigDecimal`s, this operation will round towards 0, which is to say, it will not return any fractional digits.

```js

const expected = 4n / 2n;
// ↪ 2n

const rounded = 5n / 2n;
// ↪ 2n, not 2.5n

```

[See the advanced documentation for use with bitwise operators.]()

### Comparisons

A `BigInt` is not strictly equal to a `Number`, but it is loosely so.

```js

0n === 0
// ↪ false

0n == 0
// ↪ true

```

`Number`s and `BigInt`s may be compared as usual.

```js
1n < 2
// ↪ true

2n > 1
// ↪ true

2 > 2
// ↪ false

2n > 2
// ↪ false

2n >= 2
// ↪ true
```

They may be mixed in arrays and sorted.

```js

const mixed = [4n, 6, -12n, 10, 4, 0, 0n];
// ↪  [-12n, 0, 0n, 10, 4n, 4, 6]

mixed.sort();
// ↪ [-12n, 0, 0n, 10, 4n, 4, 6]
```

### Conditionals

A `BigInt` behaves like a `Number` in cases where it is converted to a `Boolean`: `if`, `||`, `&&`, `Boolean`, `!`.

```js

if (0n) {
	console.log('Hello from the if!');
} else {
	console.log('Hello from the else!');
}

// ↪ "Hello from the else!"

0n || 12n
// ↪ 12n

0n && 12n
// ↪ 0n

Boolean(0n)
// ↪ false

Boolean(12n)
// ↪ true

!12n
// ↪ false

!0n
// ↪ true

```

### Other API Notes

`BigInt`s may also be used in `BigInt64Array` and `BigUint64Array` [typed arrays](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray) for 64-bit integers.

```js
const view = new BigInt64Array(4);
// ↪ [0n, 0n, 0n, 0n]
view.length;
// ↪ 4
view[0];
// ↪ 0n
view[0] = 42n;
view[0];
// ↪ 42n

// Highest possible BigInt value that can be represented as a
// signed 64-bit integer.
const max = 2n ** (64n - 1n) - 1n;
view[0] = max;
view[0];
// ↪ 9_223_372_036_854_775_807n
view[0] = max + 1n;
view[0];
// ↪ -9_223_372_036_854_775_808n
//   ^ negative because of overflow

```

[For more about `BigInt` library functions, see the advanced section.]()


## Gotchas & Exceptions

### Interoperation with `Number` and `String`

The biggest surprise may be that `BigInt`s cannot be operated on interchangeably with `Number`s. Instead a `TypeError` will be thrown. ([Read the design philosophy for more about why this decision was made.]())


```js

1n + 2
// ↪ TypeError: Cannot mix BigInt and other types, use explicit conversions

1n * 2
// ↪ TypeError: Cannot mix BigInt and other types, use explicit conversions

```

`BigInt`s also cannot be converted to `Number`s using the unary `+`. `Number` must be used.

```js

+1n
// ↪ TypeError: Cannot convert a BigInt value to a number

Number(1n)
// ↪ 1

```

The `BigInt` *can* however be concatenated with a `String`.

```js

1n + '2'
// ↪ "12"

'2' + 1n
// ↪ "21"

```

### Rounding

As noted above, the `BigInt` only represents whole numbers. `Number` only reliably represents integers up to 2^53. That means both dividing and converting to a `Number` can lead to rounding.

```js

5n / 2n
// ↪ 2n

Number(151851850485185185047n)
// ↪ 151851850485185200000

```

### Other Exceptions

Attempting to convert a fractional value to a `BigInt` throws an exception both when the value is represented as an `Number` and a `String`.

```js
BigInt(1.5)
// ↪ RangeError: The number 1.5 is not a safe integer and thus cannot be converted to a BigInt

BigInt('1.5')
// ↪ SyntaxError: Cannot convert 1.5 to a BigInt

```

`Math.fround` and `|` throw exceptions when called with a `BigInt` to preserve the current expectations of `asm.js`.

```js

Math.fround(1n)
// ↪ TypeError: Cannot convert a BigInt value to a number

1n|0
// ↪ TypeError: Cannot mix BigInt and other types, use explicit conversions

```


## About the Proposal

### Motivation: Why Do We Need Such Big Numbers?

- guids
- fs.stat
- nanoseconds!
- ability to store credit cards as numbers
- include error example with chips and incorrect arithmetic?

### Design Goals, Or Why Is This Like This?

These principles guided the decisions made with this proposal.

### Don't break user intuition

When a messy situation comes up, this proposal errs on the side of throwing an exception rather than silently giving a bad answer. This is what's behind throwing a TypeError on adding a BigInt and a Number: If we don't have a good answer, better to not give one.

Some JavaScript users will surely have the intuition that everything will just work, and that exceptions will not be thrown, when doing this sort of interoperation, even at the cost of losing precision. Axel Rauschmeyer [elaborated](https://gist.github.com/rauschma/13d48d1c49615ce2396ce7c9e45d4cd1) a proposal which elaborates this approach. [Further discussion](https://github.com/tc39/proposal-integer/issues/36) raised a few issues with this approach which are a direct result of upgrading the behavior of existing Numbers, both in terms of compatibility and reasonable expectations of invariants of JavaScript values. For now, this repository will stick with the distinct type approach.

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


### State of the Proposal

This proposal is currently in Stage 3. [Check out the TC39 stages to table to understand what that means.](https://tc39.github.io/process-document/)

`BigInt` has been shipped in Chrome and is underway in Node, Firefox, and Safari. 

