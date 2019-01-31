---
layout: post
title:  "Safe Use of bignumber.js"
date:   2019-01-31 16:12:00 +0800
categories: update
author: David Leung
---

JavaScript numbers are often inadequate to precisely represent numerical
values for many applications. `bignumber.js` is a popular JavaScript package
used in many financial and blockchain related projects to overcome this
problem.

We present some of the perils of working with `bignumber.js` and how to
mitigate them.

<!--more-->

## Arbitrary-precision arithmetic in JavaScript

JavaScript has only one number type, aptly named `Number`. `Number` supports
64-bit floating-point values and is specified in the IEEE 754 standard. This
format, while good for many general purpose applications, lacks the precision
needed for certain applications that are sensitive to rounding errors.

Some programming languages such as Clojure handle this problem by automatically
promotes numbers that require more precision to a `BigInt` or `Ratio` type.
Other languages address these needs by including arbitrary-precision number
types in their standard library. In stark contrast with these other languages,
JavaScript is probably one of the few mainstream languages without standard
support for arbitrary-precision types.

Fortunately, the vibrant JavaScript community came up with many solutions. One
of the more notable solution is `bignumber.js`.

## How to use BigNumber.js

We are not going to cover how to use `bignumber.js`. For that, you can refer to
its [excellent documentation](http://mikemcl.github.io/bignumber.js/).

## How not to use BigNumber.js

`BigNumber` is a great library and most of the time, it doesn't get in your
way. Because it's so easy to use, it's equally easy to fall under the illusion
that all your calculations are now precise and *safe*.

For our purpose, safety here means `BigNumber` is able to give correct result
in numerical computations, and that the serialized form of a `BigNumber` can be
parsed by other libraries that expect a (potentially big) number without error.

### Don't Use `BigNumber` in Primitive Operations

When `BigNumber` instances are used in primitive operations such as `+`, `>`,
*type coercion* occurs. The primitive operator will cause its operands, in our
case `BigNumber`, to be automatically converted to a primitive type by calling
the `valueOf` method on the object. The result of the operation may appear
correct superficially, but semantically incorrect.

For example, the following is a mathematically false statement:

```
0.1 > 0.3 - 0.2
```

When expressed as BigNumber, the result we obtain is also correct:

```javascript
BigNumber(0.1).isGreaterThan(BigNumber(0.3).minus(BigNumber(0.2)))
// => false
```

If the same operation was carried out with primitive operation, the result is
incorrect:

```javascript
BigNumber(0.1) > (BigNumber(0.3) - BigNumber(0.2))
// => true
```

It is easy to accidentally use BigNumber in primitive operations. While
TypeScript catches the use of BigNumber in +, -, *, and / during compile time,
the safety guaranteed is not available at runtime.

The author of `bignumber.js` recommends making the following change to the
prototype of the default BigNumber constructor to disable *type coercion*,
which is what makes BigNumber, a non-primitive type, behaving like a primitive
type when used in the position of primitive operands.

```javascript
BigNumber.prototype.valueOf = function () {
  throw Error('valueOf called!')
}
```

Note that this only works with `BigNumber.js` version 8 or above. See
[type coercion](http://mikemcl.github.io/bignumber.js/#type-coercion) in the
`bignumber.js` documentation for details.

### Don't Use `toString()` to get a string representation of `BigNumber`

Some libraries, such as `ethers.js`, work with arbitrary-precision numbers by
parsing those numbers as `string`. The numbers are expected to be in
*non-exponential* format.

For example, those libraries would expect the maximum 256-bit unsigned integer
to be expressed as:

```
115792089237316195423570985008687907853269984665640564039457584007913129639935
```

One may reasonably expect the `toString()` method of `BigNumber` to give this
representation. Unfortunately, this is not the case:

```javascript
BigNumber(2).pow(256).minus(1).toString()
// => 1.15792089237316195423570985008687907853269984665640564039457584007913129639935e+77
```

The behavior of `toString()` mirrors that of the JavaScript primitive numbers.
That is, when a `BigNumber` the exponent part of a BigNumber is larger than 20
or smaller than -7, the exponent notation is used.

To avoid parsing error in libraries that do not work with the exponential
format, the conversion to string should be done with `toString(10)`:

```javascript
BigNumber(2).pow(256).minus(1).toString(10)
// => 115792089237316195423570985008687907853269984665640564039457584007913129639935
```

Similar to the use of `BigNumber` in primitive operations, this mistake is also
easy to make. This problem cannot be reliably prevented, however, as it is not
possible to disable this conversion behavior at this time short of using the
`.toString(10)` method above.

To avoid the exponential notation becoming a problem in practice, we can
configure the `BigNumber` constructor to almost never convert those `BigNumber`
instances to exponent notation by changing the `EXPONENTIAL_AT`. This is done
by changing the `EXPONENTIAL_AT` setting from its default `[-7, 20]` to the
maximum allowed range `[-1e9, 1e9]`

See
[http://mikemcl.github.io/bignumber.js/#exponential-at](http://mikemcl.github.io/bignumber.js/#exponential-at)

## TL;DR

To use `BigNumber` safely, create a new `BigNumber` constructor that prevents
its instances from being use in primitive operations and converting to string
in exponent format:

```javascript
const SafeBigNumber = BigNumber.clone({
  EXPONENTIAL_AT: [-1e9, 1e9]
})

// Prevent use in primitive operations.
// See https://mikemcl.github.io/bignumber.js/#type-coercion
SafeBigNumber.prototype.valueOf = function() {
  throw Error('Conversion to primitive type is prohibited')
}
```

This create `BigNumber` that are compatible with the ones created by the
default `BigNumber` constructor.

You can now use an instance of `SafeBigNumber` just like a regular `BigNumber`:

```javascript
const one = new SafeBigNumber(1)
BigNumber.isBigNumber(one)          // => true
```

## Author
[David Leung](https://github.com/dhl)
