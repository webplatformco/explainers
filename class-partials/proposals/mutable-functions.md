# Mutable Functions

## Motivation

Many [partials](..) use cases require the ability to mix in partials to an existing class, which is why it’s listed as a [core requirement](../#it-should-be-possible-to-apply-partials-to-an-existing-class).

[Protocols](../prior-art.md#first-class-protocols-proposal) allow post-hoc extension, but treat all naming conflicts as errors, while [subclass factories](../prior-art.md#subclass-factories-mixins) sidestep this issue by piggybacking on inheritance.

In JS, functions are currently **immutable** in terms of logic.
While the function objects themselves are mutable, the function's logic is not.
There is no way to intercept calls to a function and add side effects or override return values without creating a new function.

However, for many use cases the ability to add side effects to **existing methods** without breaking references to them, is crucial.
This includes all cases where **functions are used as lifecycle hooks**, such as all the web components use cases.
While it could be argued that ideally, a pub/sub mechanism would be more suitable and naturally composable for many of these use cases, this doesn't change the fact that it is a widely used pattern.
And it could be argued that effectively, OOP inheritance is also an expression of this pattern: authors can schedule initialization logic to run at instance creation time by adding it to a method with a particular name.

## Design space

There are two main ways to add side effects to existing functions:
1. In-place (preserving references)
2. By creating a new function which can be extended with side effects.

Neither of these needs to be specific to partials, depending on the design, they can be implemented as broader features of `Function` objects.

Below is a detailed exploration of the design space, from the least controversial to the most controversial design decisions.

#### Side effect context and arguments

It seems reasonable that side effects would be called with the **same context and arguments as the original function body**.

#### Restricting side effects to void methods

Given that the primary use case for extending existing methods is adding side effects to lifecycle hooks, it appears that it is probably acceptable to simply **ignore return values**, resolving one of the big open questions around designs that allow function composition.

#### Side effect execution order

Another question is **_when_ are side effects executed?**

Some things are obvious:
- Side effects should be executed in the order they were added.
- You shouldn't be able to add the same side effect twice.

But are they executed **before or after the original function body?**
Or do we expose ways to do either?

Ideally, side effects should be independent, both from each other, and from the original function body, so it seems counter-intuitive to expose too much control over the execution order.
That said, there is this common pattern:

```js
class B extends A {
	// ...
	foo() {
		super.foo();
		// ideally we want side effects to be executed here
		// (foo body)
	}
}
```

If we go with the conceptual model where partials sit between the implementing class and the superclass, then side effects should be executed before the original function body, but after the superclass method.
But how to define this more broadly, given that the super method can be called at any point?
That seems like a can of worms best avoided.

However, if side effects are executed _after_ the function body, then authors can always add the side effects to the super method, checking the instance's prototype chain:

```js
A.prototype.foo.addSideEffect(function () {
	if (this instanceof B) {
		// ...
	}
})
```

Running side effects after the function body may also make it easier to explain why their return values are ignored.

#### Can side effects be removed and/or introspected?

Another open question is around **introspection**.
To what extent can side effects be inspected and/or removed?

On one side of the spectrum, we could have side effects be a public array of functions that can be manipulated at will.

```js
let a = () => { console.log('a'); }
let sideEffect = () => { console.log('side effect'); }
a(); // logs 'a'
a[Function.sideEffects].push(sideEffect);
a(); // logs 'side effect' and then 'a'
a[Function.sideEffects].splice(0, 1);
a(); // logs 'a'
```

This provides maximum flexibility, but removes all guarantees.
Essentially, all bets are off.
Even if a given partial is applied, there is no guarantee that its side effects will be executed.
At the very least the property should be non-writable so that authors cannot simply overwrite the array.

On the other side of the spectrum, there is total opacity:
side effects are added through a method, and cannot be inspected or removed,
they are effectively swallowed by the function and there is no return from there (pun not intended).

```js
a(); // logs 'a'
a.addSideEffect(sideEffect);
a(); // logs 'side effect' and then 'a'
```

An intermediate approach might be something like a `WeakSet` of functions:

```js
let a = () => { console.log('a'); }
let sideEffect = () => { console.log('side effect'); }
a(); // logs 'a'
a[Function.sideEffects].add(sideEffect);
a(); // logs 'side effect' and then 'a'
a[Function.sideEffects].delete(sideEffect); // need reference to remove
a(); // logs 'a'
```

Then, mutating side effects requires a reference to them, restricting destructive operations.
This also ensures that the same side effect cannot be added twice without additional logic.
Devtools can always wrap the method to log calls so that it can trace methods appropriately.

#### Return values

TBD

#### In-place vs immutability-preserving

That is probably the hairiest part of the design space.
The ability to add side effects in place, to any function, without breaking references to it can be incredibly powerful for a number of use cases extending beyond class partials and allows for class partials that are minimally invasive.

It does break assumptions around immutability, but per the [Priority of Constituencies](https://www.w3.org/TR/design-principles/#priority-of-constituencies), philosophical purity is secondary to author needs.

But if in-place side effects are too controversial, another idea that preserves immutability is to restrict the ability to have a mutable list of side effects to a **special function type** which can be created directly through a special keyword but is also created implicitly when trying to add side effects to a non-mutable function:

```js
mutable function foo () {
	console.log('foo');
}
```

This can be combined with other keywords (e.g. `async`) to create mutable functions of different types.

whose creation follows the [Multiton pattern](https://en.wikipedia.org/wiki/Multiton_pattern),
i.e. each non-mutable function corresponds to a single mutable function that can be extended with side effects.
Adding side effects is done through a call to a memoized `Function` method, e.g. `Function.prototype.addSideEffect()` which looks up and creates the mutable function if it doesn't exist, and then adds the side effects to it.

While this is not as unobtrusive as in-place side effects, it's the second best option:
- References only ever break once, not every time a side effect is added
- Functions can even be defensively defined as extensible so that references never break

Adding side effects to a function would be done through a call to a memoized `Function` method, e.g. `Function.prototype.addSideEffect()`.
The first time the method is called for a given function, a new such function is created that wraps the original function, adds the side effects, and returns it.
Future calls to `addSideEffect()` for the same function will add side effects to the same wrapped function, and return it.

This special type of function can be:
1. A new `Function` subclass (e.g. `MutableFunction`)
2. A `Proxy` object that wraps the original function. Slower [^1], but more transparent.

[^1]: Is it? [This benchmark](https://jsbenchmark.com/#eyJjYXNlcyI6W3siaWQiOiJ4a3BUc1NaOEt0THo4VWR3MzduX2MiLCJjb2RlIjoiREFUQS5iMSgpIiwibmFtZSI6IlBsYWluIGNhbGwiLCJkZXBlbmRlbmNpZXMiOltdfSx7ImlkIjoicDBpUTFnSlExOWd3ZnNITEF1OElVIiwiY29kZSI6IkRBVEEuYjIoKSIsIm5hbWUiOiJQcm94eSBjYWxsIiwiZGVwZW5kZW5jaWVzIjpbXX1dLCJjb25maWciOnsibmFtZSI6IkJhc2ljIGV4YW1wbGUiLCJwYXJhbGxlbCI6dHJ1ZSwiZ2xvYmFsVGVzdENvbmZpZyI6eyJkZXBlbmRlbmNpZXMiOltdfSwiZGF0YUNvZGUiOiIvLyBtYWluIGZ1bmN0aW9uXG5sZXQgZm4gPSBmdW5jdGlvbihhKSB7IHJldHVybiBhICsgMSB9XG4vLyBzaWRlIGVmZmVjdFxubGV0IHNlID0gZnVuY3Rpb24oYSkgeyBjb25zb2xlLmxvZyhhKSB9XG5cbmxldCBiMSA9IGZ1bmN0aW9uKC4uLmFyZ3MpIHsgXG4gIGxldCByZXQgPSBSZWZsZWN0LmFwcGx5KGZuLCB0aGlzLCBhcmdzKTtcbiAgZm9yIChsZXQgcyBvZiBiMS5zZSkgUmVmbGVjdC5hcHBseShzLCB0aGlzLCBhcmdzKTtcbiAgcmV0dXJuIHJldDtcbn1cbmIxLnNlID0gW3NlXTtcblxubGV0IGIyID0gbmV3IFByb3h5KGZuLCB7IFxuICBhcHBseSh0YXJnZXQsIHRoaXNBcmcsIGFyZ3MpIHtcbiAgICBsZXQgcmV0ID0gUmVmbGVjdC5hcHBseSh0YXJnZXQsIHRoaXNBcmcsIGFyZ3MpO1xuICAgIFJlZmxlY3QuYXBwbHkoc2UsIHRoaXNBcmcsIGFyZ3MpO1xuICAgIHJldHVybiByZXQ7XG4gIH1cbn0pO1xuXG5yZXR1cm4ge2IxLCBiMn07In19) doesn't seem that definitive at all.

Then, the class partial syntax will simply take care of calling this method and replacing any extended functions with their wrapped versions.
If the implementing class does not include a base method for the function extended, a new dummy method  will be created.
This is important, as we cannot expect implementing classes to implement very possible lifecycle hook.
E.g. a web components partial may want to run code in `adoptedCallback()`, which is relatively rare for most web components to implement.

While replacing references may be acceptable for class methods, it is not acceptable for constructors, as that effectively breaks the reference to the class.
However, since constructors are generated anyway, perhaps they could be generated to be side effect permitting functions.
This would also allow authors to access the constructor logic function separately from the class itself, which is something that is not possible today.
This would rule out proxies as an option, for obvious compat and performance reasons.
