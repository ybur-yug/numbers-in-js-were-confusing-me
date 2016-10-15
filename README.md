# So I Wondered Something

## An Introduction
The other day, someone one twitter shared a code snippet from the V8 JavaScript engine.
It was nothing particularly exciting; simply part of the logic for how it handles sorting in arrays.
What intrigued me is that there was a specific cutoff in which they did not do a quicksort algorithm on the array, and rather had some customized logic that handled other array sizes within two bounds.
I believe they were 22 (the one we are looking at in this window), and something like >= 2000.
The part that intrigued me was 22.
Programmers almost always tend to pick powers of 2 when selecting arbitrary fill-in values in which a general range would be okay.
This almost definitely had to be there because of some benchmark that was worth attaining; which is perfectly sensical for a JavaScript engine.

The particular block that caught my eye is [this](https://github.com/v8/v8/blob/ca6e40d7ba853319c15196fef3f4536c8b3929fe/test/mjsunit/array-sort.js#L423):

```javascript
function TestSortDoesNotDependOnArrayPrototypePush() {
  // InsertionSort is used for arrays which length <= 22
  var arr = [];
  for (var i = 0; i < 22; i++) arr[i] = {};
  Array.prototype.push = function() {
    fail('Should not call push');
  };
  arr.sort();

  // Quicksort is used for arrays which length > 22
  // Arrays which length > 1000 guarantee GetThirdIndex is executed
  arr = [];
  for (var i = 0; i < 2000; ++i) arr[i] = {};
  arr.sort();
}
```

Now, this is a little interesting just because of the implementation details. 
Okay, so we have something we do for super short arrays, and something else for giant ones.
I spelunked a bit more looking at sorting stuff and found another interesting tidbit that ended up sending me down a wormhole.
You can see it [here](https://github.com/v8/v8/commit/57415753#diff-cd88fdcb48e2c895640a630ca5b05ac2R369), and boy did it sent me through a loop.

```javascript
var arr = [o(1), o(2), o(4), o(8), o(16), o(32), o(64), o(128), o(256), o(-0)]
```

A list of an array length `-0`.
This isn't what this code is doing it all but it just got me thinking about some implementation details.

## Investigating
Let's open the console and poke around with `-0`.

```javascript
> -0 === 0
true
> -0 == +0
true
> -0 === +0
true
> -0 / 0
NaN
> -0 / 1
-0
> 1 / -0
-Infinity
> 1 / +0
Infinity
> Object.is(-0, +0)
false
> Object.is(-0, -0)
true
> Object.is(-0, +0)
false
> Object.is(-0, 0)
false
> +0 == 0
true
> +0 === 0
true
> Object.is(+0, 0)
true
```

So there are a few layers here.
Let's break this down into a few chunks that we can cherry-pick through:

```javascript
// so is -0 the same as 0?
> -0 === 0
// seems to be so...what about 0?
true
// okay, cool, what about +0?
> -0 == +0
true
// neat, so wtf is the difference then? why do we have this?
> -0 === +0
true
// well it seems this is just all the same
// is +0 like this too?
> +0 == 0
true
> +0 === 0
// does it break division???
> -0 / 0
// ok...
NaN
> -0 / 1
// ok...
-0
// so do we break division in the expected manner?
> 1 / -0
-Infinity
// seems to be so...
> 1 / +0
Infinity
> 1 / 0
Infinity
// one last test
> 0 / 0
Nan
> +0 / 0
NaN
> -0 / 0
NaN
```

So what we can gather here is that +0 and -0 seem to behave in quite the similar way.
It appears these zeroes all just look and act alike.

## `Object.is`
However, in the new world of JS as of ES2015 there was a new function intoduced called `Object.is`.
`Object.is` is a function that takes two arguments and simply tells you if they are **really** the same.
The function returns true if one or more of the following is correct.

```text
- both undefined
- both null
- both true or both false
- both strings of the same length with the same characters
- both the same object
- both numbers and
  - both +0
  - both -0
  - both NaN
  - or both non-zero and both not NaN and both have the same value
```

Now, why is this?
I really don't know yet, I'm still investigating.
But I have some fundamental questions I would like to understand that are coming of this.
The terminology `Object.is` is quite simple.
It seems to me that a function extending from `Object` would reasonably only deal with objects.
However, if we dig in a bit further it quickly becomes clear it is not.
From [the Mozilla Developer Network page](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is) we can dig in even further:

```text
This is not the same as being equal according to the == operator.
The == operator applies various coercions to both sides (if they are not the same Type) before testing for equality (resulting in such behavior as "" == false being true), but Object.is doesn't coerce either value.

This is also not the same as being equal according to the === operator.
The === operator (and the == operator as well) treats the number values -0 and +0 as equal and treats Number.NaN as not equal to NaN.
```

So this got me thinking even further.
I wanted to start off understanding why `Object.is` behaves differently, but I began to think of something else.
JavaScript has 6 types of primitives: `Boolean`, `String`, `Null`, `Undefined`, `Symbol` and `Number`.
A primitive isn't an `Object`, is it?
I took a look at the official specification:

```text
4.3.2 primitive value
member of one of the types Undefined, Null, Boolean, Number, Symbol, or String as defined in clause 6.

NOTE:
A primitive value is a datum that is represented directly at the lowest level of the language implementation.
```

Now, what if we look at the same specification for `Object`?

```text
4.3.3 object
member of the type Object

NOTE:
An object is a collection of properties and has a single prototype object.
The prototype may be the null value.
```

And from here it only makes sense to see the official definition for a Prototype

```text
4.3.5 prototype
object that provides shared properties for other objects
NOTE:
When a constructor creates an object, that object implicitly references the constructor's prototype property
for the purpose of resolving property references. The constructor's prototype property can be referenced by
the program expression constructor.prototype, and properties added to an object's prototype are shared,
through inheritance, by all objects sharing the prototype. Alternatively, a new object may be created with an
explicitly specified prototype by using the Object.create built‑in function.
```

So, let's break this down:

- Primitives are the simplest thing. They are defined at `the lowest level of the language implementation`. I'm not sure what that means yet.
- I know that primitives in other languages have a fixed allocation of memory. Java is a good example of this.
- Objects can be the null value, and this is a primitive. So maybe a primitive is an `Object`?
- All primitives arent the same though, so `null` might be the exception and not the rule
- It seems prototypes make everythingA
- Null might sort of be an object but none of the other primitives (including the set of **all** numbers in JavaScript) are not, seemingly

So when I really started to think about this it seemed that it was just quite strange that `Object.is` would take a primitive as an argument because primitives, outside `null`, from what I have seen thus far are not at all in fact `Object` types,
However, I thought maybe next it was best I ignore this and instead try to grok `-0` and `+0` even if there is another weird thing I want to investigate.

## Numbers In JavaScript (Specifically,`0` and its cousins)
This is where it starts to get interesting.
If we poke through our index we get 3 sections back to back that just the title can answer some questions for us.

```text
4.3.20 Number value
primitive value corresponding to a double‑precision 64‑bit binary format IEEE 754‑2008 value

Note:
A Number value is a member of the Number type and is a direct representation of a number.

4.3.21 Number type
set of all possible Number values including the special “Not‑a‑Number” (NaN) value, positive in伀氂inity, and negative in伀氂inity

4.3.22 Number object
member of the Object type that is an instance of the standard built‑in Number constructor

Note:
A Number object is created by using the Number constructor in a new expression, supplying a number value as
an argument. The resulting object has an internal slot whose value is the number value. A Number object can
be coerced to a number value by calling the Number constructor as a function (20.1.1.1).
```

So here we have it, an answer as to if we can consider `Object.is` to be intaking a primitive or `Object` type, as its right here in the spec.
It seems that a number value is something like `1` or `3` or `-5`.
However, a `Number object` is something we make a bit differently:

```javascript
> new Number
[Number: 0]
> new Number(-0)
[Number: -0]
> new Number(+0)
> [Number: 0]
```

So these are pretty clearly objects.
The main reasoning is that this is clearly a key : value pair and that is the core of the `Object` type in JavaScript (unless its `Null`, as we learned earlier).
What makes this different, if at all outside this?
Well, we could find out more by looking at the constructor information at 20.1.1.1, but let's toy around a bit first:

```javascript
> one = new Number(1)
[Number: 1]
> one == 1
true
> one === 1
true
> b = one['Number']

> b == null
true
> b === null
false
> Object.is(b, null)
false
> Object.is(one, 1)
false
```
So now we can see some new stuff.
The instance of the number Object strictly equals the primitive Number value 1.
However, `Object.is` declares them as disparate items.
This isn't quite the behaviour we had earlier but it gives us another track to go down.


