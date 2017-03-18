# Numbers, JavaScript.

## Introduction
In JavaScript, a number of any sort is typically considered a `primitive`.
This generally is the simplest unit of a language.
The official ECMAScript 2016 specification says this in its definitions:


> 4.3.2 primitive value
> member of one of the types Undefined, Null, Boolean, Number, Symbol, or String as defined in clause 6.
> NOTE:
> A primitive value is a datum that is represented directly at the lowest level of the language implementation.

There are also some subtleties to numbers in JavaScript, primarily because it turns out that rather than having a dedicated concept of an Integer, JavaScript only has floating points, as defined per the IEEE 754 Double Precision Float specification.

#### Not-Quite-Typical Points: 1

One of the interesting consequences of all numbers following this specification is the presence of signed zero.
The specification requres both `+0` and `-0` be defined. `1 / -0` should be `-infinity` and `1/+0` should be `+infinity`, while being undefined for `+0 / +0` and `+infinity / -infinity`.
Let's open up a shell and examine this.

```javascript
> -0 === 0
true
> -0 == 0
true
> -0 === +0
true
> -0 == +0
true
> +0 === -0
true
> +0 == 0
true
> +0 === 0
true
> +0 == -0
```
Okay, but what about our division rules?

```javascript
> 1 / -0
-Infinity
> 1 / +0
Infinity
> +0 / +0 
NaN
> 0 / 0
NaN
> 0 / -0
NaN
> -0 / -0
NaN
```

Everything fits.
However, beyond `==` and `===`, there is another way in JavaScript to strictly compare two things, called `Object.is`.
Let's see what it thinks of how these zeroes compare:

```javascript
Object.is(0, 0)
true
Object.is(0, -0)
false
Object.is(0, +0)
true
```

#### Not-Quite-Typical Points: 2

Now this is a bit interesting. `0` and `-0` are not considered equal by this setup.
But what exactly _is_ `Object.is`?
It was added in ES2015, and we can find the details for this in the official spec as well.
The documentation says one or more the following conditions being `true` will make it return `true`.


> - both arguments are Undefined
> - both arguments are Null
> - both arguments are True or both False
> - both arguments are String primitives of the same length with the same characters
> - both arguments are the same Object
> - both arguments are Number types AND
>     - both arguments are +0
>     - both arguments are -0
>     - both arguments are NaN
> - or both arguments are non-zero and both not NaN and both have the same value

Okay, this seems reasonable and fits what we had.
We dont even have to go in the _why_ of this, because something is quite strange here.
Why would a function `Object.is`, _if a number is truly a primitive_, do _anything_ except throw a type error of some sort?
A primitive is not an object per the specification's definition, seemingly, as an object is certainly _not_ one of the types `Undefined, Null, Boolean, Number, Symbol, or String`.

#### Not-Quite-Typical Points: 3


