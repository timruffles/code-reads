# Code read of Ratio.js

<p class="author">Code read of <a href="https://github.com/LarryBattle/Ratio.js">Larry Battle's Ratio.js</a> by <a href="http://truffles.me.uk">Tim Ruffles</a> (<a href="http://twitter.com/timruffles">@timruffles</a>)</p>

## Contents

1. [Correctness](#correctness)
1. [Style](#style)
1. [Refactoring](#refactoring)
1. [Documentation](#documentation)


<a id=correctness></a>

## Correctness

As far as I can see `Ratio`'s numeric components are correct: there's a strong test suite and the alogrithms are based on another library.

### Parsing

`Ratio.parse("1 // 2") == Ratio.parse(0,1)` - as `/(\d|Infinity)\s*\//` matches `1 /.*`. Probably worth checking we can handle the rest of the input.


<a id=style></a>
## Style

### Naming

`divSign` probably shouldn't be abbreviated, as nothing else in the API is.

`reduce()` might be better named `simplify()` - only as `Array.reduce` etc is common in JS.

[parseToArray](https://github.com/LarryBattle/Ratio.js/blob/ba0234983b3bb136036e70955f87d21a34133ec6/lib/Ratio-0.3.11.js#L184) is hard to read - `j`, `arr` are doing a lot of different things. Since you understand hoisting, I'd just define descriptive vars in each of the branches. You could even break it into a lookup object of methods to keep each one clean and isolated.

```
	toArray: {
		number: function() {…},
		decimal: function() {}
	}
```

and use it in `parseToArray`

```
	parseToArray: function(obj) {
		var type = Ratio.guessType(obj);
		var handler = Ratio.toArray[type];
		if(!handler) return [NaN,1];
		return handler(obj);
	}
```

`RE2_RE1AtEnd` etc have2 a different naming schema from the rest of the `camelCased` code.

### Code

I found

```
var a,
b,
c;
```

hard to read as it blurs into the rest of the code, and it's not a style I've seen before. I'd indent the rest of the vars to where they'd be with one `var` per line, or actually use one `var` per line.

### Magic numbers

Might be worth linking to the ECMA spec to explain numbers like `Math.pow(2,53)`.

<a id=refactoring></a>
## Refactoring

### Immutability

`Ratio` should be made immutable. It's a [value object](http://en.wikipedia.org/wiki/Value_object) like Javascript's `Number`. Value objects are identified by their value, not their identity (other examples are Dates, Sets, Intervals etc). We don't have 'a' 1, we have 1. Being able to `1.setValue(2)` would not be helpful; equally it's unhelpful to be able to change the numerator of a `Ratio`. If we have an immutable object we can be sure it'll never be altered. Currently if we share a `Ratio` with two parts of our code, one may call `correctRatio()`, or worse `ratio.numerator =`, and we're in trouble.

This could be achieved via making all fields, e.g. `numerator`, `denominator` etc, non-writable properties via `defineProperty`. All methods that currently alter any field on `Ratio` should be changed to return a new `Ratio` similar to the implementations of `add()`.

Rather than setting `divSign` it's probably simpler just to pass it to a `formatString()` method, or make `toString()` accept an argument. Along with `alwaysReduce`, it doesn't feel like part of a `Ratio` value, it's just how one is represented.

### AMD

Might be useful to detect AMD and set a `define()` call for Ratio, so it can be used with AMD loaders.


<a id=documentation></a>
## Documentation

The documentation is clear. It would be worth explaining where Ratio will help you out above and beyond using Javascript's `Number`, and at what point it can't achieve precision.

### Parsing

Mention `isNaN()` to check whether `Ratio` could parse something.

### Comparisons

`==` actually returns false for comparing [any two Object values](http://www.ecma-international.org/ecma-262/5.1/#sec-11.9.3), not even attempting to compare them. It's `===` that compares two Objects by [checking if they refer to the same object](http://www.ecma-international.org/ecma-262/5.1/#sec-11.9.6).

```
However, equalivance( == ) will compare the object and not the value of the object. Use .equals() instead.
```

### Use === not == in method documentation

Just to be very explicit that you mean `=== false` not `== false` (which matches rather a lot, e.g. `"\t\t\t\t\t\n\n\n    \t"` etc).
