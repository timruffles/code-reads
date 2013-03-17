# Code read of Gravy.js

<p class="author">Code read of <a href="https://github.com/muffs/Gravy/">Julian Connor's gravy.js</a> by <a href="http://truffles.me.uk">Tim Ruffles</a> (<a href="http://twitter.com/timruffles">@timruffles</a>)</p>

## Contents

1. [Correctness](#correctness)
1. [Style](#style)
1. [Refactoring](#refactoring)
1. [Documentation](#documentation)


<a id=correctness></a>

## Correctness

### Modifiying Backbone.View

The documentation suggests `Backbone.View` and `Gravy` are different things; the code doesn't achieve this. Use `Backbone.View.extend()` to define Gravy and avoid modifying the `Backbone.View` prototype (in pull request).

### _v

`_v` isn't required as far as I can see - it's set in `validateAll` and used to avoid firing clear callbacks in `validate`. However `validate` is not called by `validateAll` - `_validateNode` is. It's therefore not required. Had it been given an intention revealing name - as suggested in the [naming](#naming) section - such as `_validatingAll` this probably would have been spotted.

### Globals

There were two accidental global variables created in `for in` loops (in pull request).

### Out of date comments

In `_validateNode` this comment is out of date (no early return occurs), as are a few of the other comments.

```javascript
/*
* If value is optional, return here.
*/
```

### Unused vars

See [manual hoisting](#manual).

<a id=style></a>

## Style

`if` statements should be reserved for logic; if assignment is done, it should be the only contents of the `if` condition (`if(current = current.parentElement)`). Doing otherwise makes for very complex code.

```javascript
/*
 * Checks View and Model for validation method.
 */
if ( !( (validator = validator.validator) &&
                      (validator = this[validator] || this.model[validator])) )                        
```

The above has no less complexity than the more explicit code below, it simply hides it in short circuiting and an overloaded if. The following code is the same to the interpreter, but far more readable to people. In addition, the clarity likely renders the comment superflous.

```javascript
var validatorName = validator.validator;
validator = this[validatorName] || this.model[validatorName];
if (!validator)
    throw new Error("[Gravy] Unable to find validator for: " + name);
```



### Call me maybe

If you're using apply with a known number of args, use `call` instead, it's much clearer. `apply` suggests a scenario with an unknown number of arguments to handle.

`fn.apply(ctx,[fixedArgA,fixedArgB])`

`fn.call(ctx,fixedArgA,fixedArgB)`

### Assertions

Assertions clean up your code and reveal intention.

```javascript
/*
* Throw error if callback is not a function and Gravy was unable to
* find callback
*/
if ( !_.isFunction(callback) && !(callback = this[callback]))
    throw new Error("[Gravy] Unable to find callback: " + callback);
```

vs

```javascript
callback = _.isFunction(callback) ? callback : this[callback];
assertFunction(callback,"Unable to call callback");
```

<a id="naming"></a>

### Naming

Library code should have good, long, intention revealing names. Even in application code there's almost never a reason to abbreviate anything beyond `i`, `k` and `v` and perhaps `attrs` and `opts`.

The names `_v`, and `_r` obscure their purpose, possibly leading to `_v` existing when not required. I suggest `_reservedNames` and `_validatingAll`.

`validateNode()` is really `validateAttribute()` - it receives a value not a DOM Node.

Throughout `node` is used where `element` would be preferred. `Element` is a more specific word for the type of DOM node we're talking about - nodes are a [superclass](https://developer.mozilla.org/en-US/docs/DOM/Node) shared with text nodes, document fragments etc. We're only ever validating DOM `Element`s, so it's better to chose `element` as a specific name that removes any doubt.

<a id=manual></a>

### Manual variable jungle

Manual hoisting can lead to confusion in long functions if a single `var` statement is used. If you have a lot of variables required for a long function, it's likely more maintainable to have one var statement per line. This makes them more obvious, so in refactoring unused vars will hopefully get spotted. It also leads to cleaner diffs (otherwise adding and removing vars always modifies the same line, making git blame fairly useless).

Probably as a result of the poor visibility, we have unused vars in `validate`: `error = null, success = null`.

### Reserved Words

If `_r` is private, why worry about it changing? If we're not worried we can stick with direct access (`this.clear` vs `this[this._r.clear]`) and keep `_r` as a lookup object for testing whether an attribute is reserved.

Further, if we can be confident that the private values will not change, for this lookup role we can replace `this._reserved(word)`, an O(N) loop, with `word in this._r` which is an O(1) lookup.

### null with purpose

Whenever I see `null`, I wonder why we're using it instead of `undefined`. Unless there's a specific use of `null` (and I can't see it in the code aside from initialising vars to null), leave variables as `undefined`.

### Comments are for "why"

Comments are for "why" if something is confusing, suprising or non-obvious. They're not for "what" or "how" - that's what the code is describing. Use intention revealing names in preference to comments for self documenting code.

<a id=refactoring></a>

## Refactoring

### Isolate complexity

The various ways of specifying a callback should't have to create complexity throughout the codebase. Isolate it in a `getCallback(potentialCallback,"callbackName")` function that either takes the given function, or looks for it in the view or model. This would reduce repetition and increase maintainability (eg ease of adding/changing lookup locations).

### Finish the job

The work going on on these lines is left unfinished, which leads to more work perculating down the function, from:

```javascript
if ( !(validator = vKey instanceof Object ?
       vKey : (this[vKey] || this.model[vKey])) ) 
```

to:

```javascript
result  : (!val && optional) || validator.apply(!!this[vKey] ? this : this.model, [val]),
```

Adding this immediately below the line completes the work this part of the code has - giving the rest of the code a clean 'API' to call - a simple one argument validator function.

```javascript
validator = _.bind(validator,!!this[vKey] ? this : this.model);
```

### Duplication

Repeated blocks like the following suggest a function should be extracted.

```javascript
if ( valid &&
       !((callback = submit.success) &&
         (callback = _.isFunction(callback) ? 
          callback : this[submit.success])) )
```

The three times rule is an effective way of keeping DRY - once you see a pattern or code shape 3 times, move it into a function.

### Give back unwrapped DOM elements

It's probably best to pass unwrapped `DOMElement`s to callbacks - using a DOM library is not something you always need to do, and it's what you expect from the rest of Backbone's event callbacks in the `event` object.


<a id=documentation></a>

## Documentation

Strong documentation, aside from the need to better justify the library.

There should be more justification for why Gravy is doing it differently from Backbone. Backbone gives you a single method for validation; calling individual invalid events for attributes is something you could easily do within it, but isn't default. Here we're giving more granuarity - this should have a rationale.

Additionally validation should probably live on the model - if not, it'd be good to see cases where we have view-specific validations that don't make sense there.

The `clear` method could be better described - it's not immediately obvious that it's called when a changed input element has a "" value.

</article>




