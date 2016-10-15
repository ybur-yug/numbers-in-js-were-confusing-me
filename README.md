# Numbers in JavaScript
## Sorry, I don't have a clever title

### Introduction
The other day, someone one twitter shared a code snippet from the V8 JavaScript engine that intrigued me.
It was nothing particularly exciting; simply part of the logic for how it handles sorting in arrays of various sizes.
For whatever reason I ended up quite interested.
What intrigued me is that there was a specific cutoff in which they did not do a quicksort algorithm on the array, and rather had some customized logic that handled other array sizes within two bounds.
I believe they were 22 (the one we are looking at in this window), and something like >= 2000.
The part that intrigued me was 22.
Programmers almost seem to always tend to pick powers of 2 when selecting arbitrary fill-in values in which a general range would be okay.
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
I spelunked a bit more looking at sorting stuff and found another tidbit that ended up sending me down the wormhole we are about to summarize and attempt to understand.
You can see it [here](https://github.com/v8/v8/commit/57415753#diff-cd88fdcb48e2c895640a630ca5b05ac2R369), and boy did it sent me through a loop at first.

```javascript
...
var arr = [o(1), o(2), o(4), o(8), o(16), o(32), o(64), o(128), o(256), o(-0)]
...
```

An array length `-0`.
Now, maybe its because I lack a pure CS background or haven't read the IEEE 754 double precision floating point specification quite closely enough, but this is something I hadn't seen before.

### Wtf is -0? Let's look at numbers in JS
Let's start by just messing around in the console

```javascript
> -0 === 0     // I always start with strict equality, because JS
true
> -0 == 0      // As expected
true
> -0 == +0     // Alright....maybe its just because `==`
true
> -0 === +0    // Well...shit
true
> -0 / 0       // Reasonable
NaN
> -0 / 1       // Okay..
-0
> 1 / -0       // as expected
-Infinity
> 1 / +0       // Normal Enough
Infinity
> +0 == 0      // Alright, the inverse of our first bit works
true
> +0 === 0     // And confirms itself.
true
```

There is nothing _really_ weird here, really.
I began by comparing `0` and `-0` strictly, just to see if there was a difference in the sense of arithmetic.
With it and the regular `==` comparison operator, it seems so far there is not a meaningful difference in the context of maths.
Now, if we have `-0` I assumed that the inverse must exist, so I tried `+0`.
I didnt _actually know_ if `+0` existed at this point but my assumption was correct.
I then thought maybe it would be different if I compared the two themselves strictly.
Maybe this would finally give me something interesting!
But nope, just another simple truth.
Next I wanted to see if division behaved properly, and it seemed to.
Last but not least was just investigating if `+0` behaved like `-0` and wasn't just equal, unsurprising.

Basically, seems like `+0` and `-0` behave just about the same.
However, there is a new thing I wanted to investigate this with that just came around in ES2015.
That would be `Object.is`, which returns true if one of more of the following conditions is met: 

- both arguments are `Undefined`
- both arguments are `Null`
- both arguments are `True` or both `False`
- both arguments are `String` primitives of the same length with the same characters
- both arguments are the same `Object`
- both arguments are `Number` types and
  - both arguments are `+0`
  - both arguments are `-0`
  - both arguments are `NaN`
  - or both arguments are non-zero and both not `NaN` and both have the same value

Let's fire up the console again:

```javascript
> Object.is(-0, +0)
false
> Object.is(-0, -0)
true
> Object.is(-0, +0)
false
> Object.is(-0, 0)
false
> Object.is(+0, 0)
true
```

So this behaves as expected.
But, if we think about JavaScript, you might remember that it has what we call the concept of `primitives`.
Now, is a `primitive` an `Object`?
Well, lets dive in...

### Primitives and Objects
The terminology with regard to `Object.is` is quite simple.
It seems to me that a function extending from `Object` would reasonably only deal with objects.
However, if we dig in a bit further it quickly becomes clear it is not, as we just saw.

```text
This is not the same as being equal according to the == operator.
The == operator applies various coercions to both sides (if they are not the same Type) before testing for equality (resulting in such behavior as "" == false being true), but Object.is doesn't coerce either value.

This is also not the same as being equal according to the === operator.
The === operator (and the == operator as well) treats the number values -0 and +0 as equal and treats Number.NaN as not equal to NaN.
```

JavaScript has 6 types of primitives: `Boolean`, `String`, `Null`, `Undefined`, `Symbol` and `Number`.
A primitive isn't an `Object`, is it?
Lets take a look at the official specification:

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
- It seems prototypes make everything
- Null might sort of be an object but none of the other primitives (including the set of **all** numbers in JavaScript) are not, seemingly

So when I really started to think about this it seemed that it was just quite strange that `Object.is` would take a primitive as an argument because primitives, outside `null`, from what I have seen thus far are not at all in fact `Object` types,

Now, what if we take these definitions and go straight to the number specification with this knowledge?

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
