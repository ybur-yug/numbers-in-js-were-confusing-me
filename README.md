# Complexity & It's Origins

## Introduction
Complexity is an inherent reality when dealing with any form of _stuff_ that is constructed.
The key word here is _constructed_, as when things are defined in very basic terms with little abstraction, complexity cannot really creep in.
Rich Hickey gives a great talk in which he brings up a definition of simple that I have found to be my favorite.
I find that understanding the opposite of complexity has been key in me understanding it in a real way.
He speaks of how the roots of the word 'simple' are `sim` and `plex`, which mean 'one twist or braid'.
This is quite easy to wrap our mind around.
If something is truly simple, it has one twist (result) to it.
A great example would be a very basic addition function:

```
def add(x, y) when is_number(x) and is_number(y), do: x + y
```

Let's look at this function.

If takes two arguments, and _if and only if_ they are both numbers, then it will add them.
It does not mutate the numbers.
It simply returns a new number and leaves those two other pieces alone to frolic as they please in the bit-forest.

In this post, we will examine something (seemingly) simple along these same lines.
We will then learn a lesson.

## An Applied example
Primitives are generally defined as something along the lines of 'the simplest possible component of a system'.
JavaScript has 6 types of primitives, in its latest form.
These are `Undefined`, `Null`, `Boolean`, `Number`, `Symbol`, or `String`.
`Number` is a good entrypoint to examine something as a whole that is simple.
Here, we will examine the `Number` primitive.

### A Starting Dive
The official ECMAScript 2016 specification says this in its definitions about `Primitive` type values:


> 4.3.2 primitive value
> member of one of the types Undefined, Null, Boolean, Number, Symbol, or String as defined in clause 6.
> NOTE:
> A primitive value is a datum that is represented directly at the lowest level of the language implementation.

There are also some subtleties to numbers in JavaScript, primarily because it turns out that rather than having a dedicated concept of an `Integer`, JavaScript only has floating point numbers, as defined per the IEEE 754 Double Precision Float specification.

#### Not-Quite-Simple Points: 1
Because everything is floats, we have the presence of a signed zero.
Signed zero just means that we have `+0`, `-0`, and `0` all defined and they have certain requirements.
The specification requres both `+0` and `-0` be defined. `1 / -0` should be `-infinity` and `1/+0` should be `+infinity`, while being undefined for `+0 / +0` and `+infinity / -infinity`.
Let's open up a shell and examine this.

To start, lets look at basic equality:

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
Awesome.

### A Bit Deeper
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

#### Not-Quite-Simple Points: 2

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

#### Not-Quite-Simple Points: 3

### Objects
So, maybe a function `Object.is` might take things that are not an Object, even though its intention to compare two objects.
The specification's definition is actually quite simple:

> 4.3.3 object
> member of the type Object
> NOTE:
> An object is a collection of properties and has a single prototype object. The prototype may be the null value.

Now, this specifically says an Object has a single `prototype` object.
This wording implies that a prototype is definitely not a primitive like a number, and is a type of object, so let's investigate it:

> 4.3.5 prototype
> object that provides shared properties for other objects
> NOTE:
> When a constructor creates an object, that object implicitly references the constructor's prototype property for the purpose of resolving property references. The constructor's prototype property can be referenced by the program expression constructor.prototype, and properties added to an object's prototype are shared, through inheritance, by all objects sharing the prototype. Alternatively, a new object may be created with an explicitly specified prototype by using the Object.create built‑in function.

Let's look at what we have learned so far.

- Primitives are supposed to be the simplets possible thing in the language, defined at the lowest level
- An `Object`'s `Prototype` may be `Null`, which is technically considered a primitive per the definitions we got from the spec
- Prototypes seem to be the root of most everything

This `Prototype` can be `Null` thing seems a bit odd, since `Null` is supposed to be a primitive and `Prototype` is stated to be an object.

#### Not-Quite-Simple Points: 4

It turns out, after diving into the `Prototype` spec, we can find another interesting tidbit:

*There is a `Number` object*

#### Not-Quite-Simple Points: 5

### The Number Object
### Prototypes and Constructors
### Go to last part of current doc
### Go into Number constructor

## Conclusion
We didn't even _really answer_ all the questions that were just brought up in that entire dive.
If you're like me, you're probably quite confused and have even more questions jotted down.

What we did was look at a `Primitive`.
These are the means of construction in the language.
By diving into the true identity of something that is the most basic building block, we have unraveled a web of observations, idiosyncrasies, and new pieces of knowledge to work off of.
This isn't because JavaScript is bad, or because Brendan Eich really is plotting our demise as software developers.
It is because complexity is a reality of systems.
Our job is to manage it.
When things get to a certain point, large decisions might be made to take new directions.
One could look at CofeeScript, TypeScript, Elm, and the myriad of other JS replacements I am not listing and conclude that there may be an issue at the origin here.
However that is beside the point.

These realities are inevitably going to be a part of systems we build.
Our job is to manage complexity, not to eliminate it.
It is a Sisyphean endeavour to try for the latter idea.
