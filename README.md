# BigInt: Arbitrary precision integers in JavaScript

Daniel Ehrenberg, Igalia. Stage 4

This proposal is complete and already merged into ECMA262 specification. See the specification text [here](https://tc39.es/ecma262/).

Thanks for help and feedback on this effort from Brendan Eich, Waldemar Horwat, Jaro Sevcik, Benedikt Meurer, Michael Saboff, Adam Klein, Sarah Groff-Palermo and others.

## Contents
1. [What Is It?](#what-is-it)
2. [How Does It Work?](#how-does-it-work)
	- [Syntax](#syntax)
	- [Example: Calculating Primes](#example-calculating-primes)
	- [Operators](#operators)
	- [Comparisons](#comparisons)
	- [Conditionals](#conditionals)
	- [Other API Notes](#other-api-notes)
3. [Gotchas & Exceptions](#gotchas--exceptions)
	- [Interoperation with `Number` and `String`](#interoperation-with-number-and-string)
	- [Rounding](#rounding)
	- [Cryptography](#cryptography)
	- [Other Exceptions](#other-exceptions)
	- [Usage Recommendations](#usage-recommendations)
4. [About the Proposal](#about-the-proposal)
	- [Motivation, Or Why Do We Need Such Big Numbers?](#motivation-why-do-we-need-such-big-numbers)
	- [Design Philosophy, Or Why Is This Like This?](#design-goals-or-why-is-this-like-this)
	- [State of the Proposal](#state-of-the-proposal)


## What Is It?

`BigInt` is a new primitive that provides a way to represent whole numbers larger than 2<sup>53</sup>, which is the largest number Javascript can reliably represent with the `Number` primitive.

```js
const x = Number.MAX_SAFE_INTEGER;
// ↪ 9007199254740991, this is 1 less than 2^53

const y = x + 1;
// ↪ 9007199254740992, ok, checks out

const z = x + 2
// ↪ 9007199254740992, wait, that’s the same as above!
```

[Learn more about how numbers are represented in Javascript in the slides from Daniel's talk at JSConfEU.](https://docs.google.com/presentation/d/1apPbAiv_-mJF35P31IjaII8UA6TwSynCA_zhfDEmgOE/edit#slide=id.g38a1897a56_0_97) 

## How Does It Work?

The following sections show `BigInt` in action. A number have been influenced by or taken outright from [Mathias Bynens's BigInt v8 update, which includes more details than this page.](https://developers.google.com/web/updates/2018/05/bigint)

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
function isPrime(p) {
  for (let i = 2n; i * i <= p; i++) {
    if (p % i === 0n) return false;
  }
  return true;
}

// Takes a BigInt as an argument and returns a BigInt
function nthPrime(nth) {
  let maybePrime = 2n;
  let prime = 0n;
  
  while (nth >= 0n) {
    if (isPrime(maybePrime)) {
      nth -= 1n;
      prime = maybePrime;
    }
    maybePrime += 1n;
  }
  
  return prime;
}

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

[See the advanced documentation for use with bitwise operators.](/ADVANCED.md)

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
// ↪  [4n, 6, -12n, 10, 4, 0, 0n]

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

[For more about `BigInt` library functions, see the advanced section.](/ADVANCED.md)


## Gotchas & Exceptions

### Interoperation with `Number` and `String`

The biggest surprise may be that `BigInt`s cannot be operated on interchangeably with `Number`s. Instead a `TypeError` will be thrown. ([Read the design philosophy for more about why this decision was made.](#design-goals-or-why-is-this-like-this))


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

For this reason, it is recommended to continue using `Number` for code which will only encounter values under 2<sup>53</sup>.

Reserve `BigInt` for cases where large values are expected. Otherwise, by converting back and forth, you may lose the very precision you are hoping to preserve.

```js
const largeFriend = 900719925474099267n;
const alsoLarge = largeFriend + 2n;

const sendMeTheBiggest = (n, m) => Math.max(Number(n), Number(m));

sendMeTheBiggest(largeFriend, alsoLarge)
// ↪900719925474099300  // This is neither argument!
```

Reserve `Number` values for cases when they are integers up to 2<sup>53</sup>, for other cases, using a string (or a `BigInt` literal) would be advisable to not lose precision.

```js
const badPrecision = BigInt(9007199254740993);
// ↪9007199254740992n

const goodPrecision = BigInt('9007199254740993');
// ↪9007199254740993n

const alsoGoodPrecision = 9007199254740993n;
// ↪9007199254740993n
```

### Rounding

As noted above, the `BigInt` only represents whole numbers. `Number` only reliably represents integers up to 2<sup>53</sup>. That means both dividing and converting to a `Number` can lead to rounding.

```js

5n / 2n
// ↪ 2n

Number(151851850485185185047n)
// ↪ 151851850485185200000

```

### Cryptography

The operations supported on `BigInt`s are not constant time.  `BigInt` is therefore [unsuitable for use in cryptography](https://www.chosenplaintext.ca/articles/beginners-guide-constant-time-cryptography.html).

Many platforms provide native support for cryptography, such as [webcrypto](https://w3c.github.io/webcrypto/Overview.html) or [node crypto](https://nodejs.org/dist/latest/docs/api/crypto.html).


### Other Exceptions

Attempting to convert a fractional value to a `BigInt` throws an exception both when the value is represented as an `Number` and a `String`.

```js
BigInt(1.5)
// ↪ RangeError: The number 1.5 is not a safe integer and thus cannot be converted to a BigInt

BigInt('1.5')
// ↪ SyntaxError: Cannot convert 1.5 to a BigInt

```

Operations in the `Math` library will throw an error when used with `BigInt`s, as will `|`.

```js

Math.round(1n)
// ↪ TypeError: Cannot convert a BigInt value to a number

Math.max(1n, 10n)
// ↪ TypeError: Cannot convert a BigInt value to a number

1n|0
// ↪ TypeError: Cannot mix BigInt and other types, use explicit conversions

```

`parseInt` and `parseFloat` will however convert a `BigInt` to a  `Number` and lose precision in the process. (This is because these functions discard trailing non-numeric values — including `n`.

```js

parseFloat(1234n)
// ↪1234

parseInt(10n)
// ↪10

// precision lost!
parseInt(900719925474099267n)
// ↪900719925474099300
```

Finally, `BigInt`s cannot be serialized to JSON. There are, however, libraries — for instance, [granola](https://github.com/kanongil/granola) — that can handle this for you.

```js
const bigObj = {a: BigInt(10n)};
JSON.stringify(bigObj)
// ↪TypeError: Do not know how to serialize a BigInt
```

### Usage Recommendations

### Coercion
Because coercing between `Number` and BigInt can lead to loss of precision, it is recommended to only use BigInt when values greater than 2<sup>53</sup> are reasonably expected and not to coerce between the two types.

## About the Proposal

### Motivation: Why Do We Need Such Big Numbers?

There are a number of cases in JavaScript coding where integers larger than 2<sup>53</sup> come up — both instances where signed or unsigned 64-bit integers are needed and times where we may want integers even larger than 64-bits.

#### 64-bit Use Cases

Often, other systems with which Javascript interacts provides data as 64-bit integers, which lose precision when coerced to Javascript Numbers.

These might come when reading certain machine registers or wire protocols or using protobufs or JSON documents that have GUIDs generated by 64-bit systems in them — including things like credit card or account numbers — which currently must remain strings in Javascript. (Note, however, `BigInt`s cannot be serialized to JSON directly. But you can use libraries like [granola](https://github.com/kanongil/granola) to serialize and deserialize BigInt and other JS datatypes to JSON.)

In node, `fs.stat` may give some data as 64-bit integers, which [has caused issues already](https://github.com/nodejs/node/issues/12115):

```js
fs.lstatSync('one.gif').ino
// ↪ 9851624185071828

fs.lstatSync('two.gif').ino
// ↪ 9851624185071828, duplicate, but different file!
```

Finally, 64-bit integers enable higher resolution — _nanosecond!_ — timestamps. These will be put to use in the [temporal proposal](https://github.com/tc39/proposal-temporal), currently in Stage 1.

#### Bigger Than 64-bit Use Cases

Integers larger than 64-bit values are most likely to arise when  doing mathematical calculations with larger integers, such as solving Project Euler problems or exact geometric calculations. Adding `BigInt` makes it possible to meet a reasonable user expectation of a high-level language that integer arithmetic will be "correct" and not suddenly overflow.

If this seems far-fetched, consider the case of [the Pentium FDIV bug](https://en.wikipedia.org/wiki/Pentium_FDIV_bug). In 1994, a bug in Pentium chips made floating point values rarely —but possibly — imprecise. It was discovered by a mathematics professor who was relying on that precision. 

### Design Goals, Or Why Is This Like This?

These principles guided the decisions made with this proposal. Check out [`ADVANCED.md`](/ADVANCED.md) for more in-depth discussion of each.

#### Find a balance between maintaining user intuition and preserving precision

In general, this proposal has aimed to work in a manner complementary to user intuition about how Javascript works. At the same time, the goal for this proposal is to add further affordances for precision to the language. Sometimes these can conflict.

When a messy situation comes up, this proposal errs on the side of throwing an exception rather than rely on type coercion and risk giving an imprecise answer. This is what's behind throwing a `TypeError` on adding a `BigInt` and a `Number` and other [exceptions detailed above](#gotchas--exceptions): If we don't have a good answer, better to not give one.

For more discussion of these choices, see [Axel Rauschmeyer's proposal](https://gist.github.com/rauschma/13d48d1c49615ce2396ce7c9e45d4cd1) and [further discussion of its effects on Numbers](https://github.com/tc39/proposal-integer/issues/36). We ended up concluding that it would be impractical to provide transparent interoperability between Number and BigInt.

#### Don't break math

The semantics of all operators should ideally be based on some mathematical first principles, to match developer expectations. The division and modulo operators are based on conventions from other programming languages for integers.

#### Don't break JavaScript ergonomics

This proposal comes with built-in operator overloading in order to not make `BigInt`s too ugly to be usable. One particular hazard, if `BigInts` were to be operated on with static methods, is that users may convert the `BigInt` into a `Number` in order to use the `+` operator on it--this would work most of the time, just not with big enough values, so it might pass tests. By including operator overloading, it would be even shorter code to add the `BigInt`s properly than to convert them to `Numbers`, which minimizes the chance of this bug.

#### Don't break the web

This proposal doesn't change anything about the way Numbers work. The name `BigInt` was chosen in part to avoid compatibility risks carried by the more general `Integer` name (and in part to make it clear that they are useful for the "big" cases).

#### Don't break good performance

Design work here has been done in conjunction with prototyping in to ensure that the proposal is efficiently implementable.

#### Don't break potential future value types extensions

When adding new primitives to the language, it is important to avoid giving them superpowers that would be very difficult to generalize. This is another good reason for `BigInt` to avoid mixed operands.

Mixed comparisons are a one-off exception to this principle, however, taken in support of the intuition design principle.

#### Don't break a consistent model of JavaScript

This proposal adds a new primitive type with wrappers, similar to `Symbol`. As part of integrating `BigInt`s into the JavaScript specification, a high amount of rigor will be required to differentiate three types floating around in the specification: Mathematical values, `BigInt`s and `Number`s.

### State of the Proposal

This proposal is currently in [Stage 4.](https://tc39.github.io/process-document/)

`BigInt` has been shipped in Chrome, Node, Firefox, and is underway in Safari.
- [V8](https://bugs.chromium.org/p/v8/issues/detail?id=6791) by Georg Neis and Jakob Kummerow.
- [JSC](https://bugs.webkit.org/show_bug.cgi?id=179001) by Caio Lima and Robin Morisset.
- [SpiderMonkey](https://bugzilla.mozilla.org/show_bug.cgi?id=1366287) by Robin Templeton and Andy Wingo.

Related specification proposals:
- [BigInt WebAssembly JS API integration proposal](https://github.com/WebAssembly/spec/pull/707)	
- [HTML serialization of BigInt](https://github.com/whatwg/html/pull/3480)	
- [BigInt as an IndexedDB key](https://github.com/w3c/IndexedDB/pull/231)
