# Class Partials

Authors: Lea Verou

> [!IMPORTANT]
> This is a work in progress. Feel free to send PRs to improve it, but this is not the time to evaluate it critically.

<details open>
<summary>Contents</summary>

1. [Use cases](#use-cases)
2. [Prior art](#prior-art)
	1. [Userland patterns](#userland-patterns)
	2. [Other languages](#other-languages)
3. [Resources](#resources)
4. [Existing proposals](#existing-proposals)
	1. [Mixins proposal](#mixins-proposal)
	2. [First-Class Protocols proposal](#first-class-protocols-proposal)
	3. [Decorators proposal](#decorators-proposal)
5. [Definitions](#definitions)
6. [Non-goals / Out of scope](#non-goals--out-of-scope)
	1. [Abstract methods](#abstract-methods)
	2. [Parameterization syntax](#parameterization-syntax)
7. [Requirements](#requirements)
	1. [Extending API surface of implementing class](#extending-api-surface-of-implementing-class)
	2. [Operate on prototypes, not instances](#operate-on-prototypes-not-instances)
	3. [Adding side effects to existing methods](#adding-side-effects-to-existing-methods)
	4. [It should be possible to apply partials to an existing class](#it-should-be-possible-to-apply-partials-to-an-existing-class)
	5. [Static reflection](#static-reflection)
	6. [Encapsulation](#encapsulation)
8. [Nice to haves](#nice-to-haves)
	1. [Single declaration of intent](#single-declaration-of-intent)
	2. [Instance reflection](#instance-reflection)
	3. [Use existing class primitives](#use-existing-class-primitives)
	4. [Reversibility](#reversibility)
	5. [Declarative class syntax](#declarative-class-syntax)
9. [Design space](#design-space)
	1. [1. Are partials syntactically distinct from classes?](#1-are-partials-syntactically-distinct-from-classes)
	2. [2. How and where is the partial included?](#2-how-and-where-is-the-partial-included)
	3. [3. Is composition distinct from inheritance?](#3-is-composition-distinct-from-inheritance)
	4. [4. Do partials operate on their own state or the full composed instance?](#4-do-partials-operate-on-their-own-state-or-the-full-composed-instance)
	5. [5. Can the partial's state be accessed from the implementing class?](#5-can-the-partials-state-be-accessed-from-the-implementing-class)
	6. [6. How to handle naming conflicts?](#6-how-to-handle-naming-conflicts)
	7. [7. Are partials inherited?](#7-are-partials-inherited)
10. [Concrete ideas (strawmans)](#concrete-ideas-strawmans)
	 1. [Function side effects](#function-side-effects)
	 2. [Syntax ideas for class partials](#syntax-ideas-for-class-partials)


</details>

This document is currently an exploration of the problem and design space, and does not yet propose any specific solution.

> [!NOTE]
> To avoid assumptions about specific patterns, this document avoids using terms like _mixins_, _traits_, _protocols_, etc. and instead uses the more general term _partials_ which is meant to encompass all of them.

## Use cases

The high-level use case for multiple inheritance is well-known: repeated logic / API surface that is not fundamental to the object’s identity but instead describes certain behaviors or traits (_has a_ rather than _is a_).

* Composable `EventTarget`
* Web Components
	* Implement HTML custom attributes that can be added on any element at any point in time
	* Browser-provided composable partials can also serve as an alternative to [`ElementInternals.type`](https://github.com/whatwg/html/issues/11061)
* ...TBD...

## Prior art

### Userland patterns

See [Prior art](prior-art.md#userland-patterns) for a detailed exploration of the existing userland patterns:
1. [Subclass factories (mixins)](prior-art.md#subclass-factories-mixins)
2. [Controllers](prior-art.md#controllers)
3. [Prototype mutations](prior-art.md#prototype-mutations)
4. [Instance mutations](prior-art.md#instance-mutations)

### Other languages

See [Prior art](prior-art.md#other-languages) for a detailed exploration of other languages.

## Resources

- [Wikipedia: Multiple inheritance](https://en.wikipedia.org/wiki/Multiple_inheritance)
- [Wikipedia: Virtual inheritance](https://en.wikipedia.org/wiki/Virtual_inheritance)
- ["Real" Mixins with JavaScript Classes](https://justinfagnani.com/2015/12/21/real-mixins-with-javascript-classes/)
- [JavaScript Mixins, Subclass Factories, and Method Advice](https://raganwald.com/2015/12/28/mixins-subclass-factories-and-method-advice.html)
- [Original paper on mixins (OOPSLA 1990)](https://www.bracha.org/oopsla90.pdf)


## Existing proposals

### [Mixins proposal](https://github.com/tc39/proposal-mixins)

Paves the cowpaths to declarativify the current pattern of superclass factories.

Pros:
- Purely syntactic sugar, so trivial to implement.

Cons:
- Affects inheritance chain, which is not always desirable.
- Needs to be specified at class definition time

### [First-Class Protocols proposal](https://github.com/tc39/proposal-first-class-protocols)

Follows the language’s current design pattern for implementing certain mixins (`Symbol.Iterator` etc).

Pros:
- Paves the cowpaths of an existing language pattern
- Allows for post-hoc extension

Cons:
- No way to extend existing methods, treats all naming collisions as errors.
- Restricted to prototype fields and methods (?)
- Author intent needs to be duplicated: once to specify the required symbol and once to call `Protocol.implement()` to apply it.

### [Decorators proposal](https://github.com/tc39/proposal-decorators)

It could be argued that class decorators are a form of partial, as one could apply a decorator to a class to add traits to it through subclassing.

```js
function addVersion(version: string) {
	return (value, { addInitializer }) => {
		// Post-define hook: add a static property at runtime
		addInitializer(function() {
			Object.defineProperty(this, "version", { value: version });
		});
	};
}

function Mixin(value) {
	// Replace the class with a subclass that adds a method
	return class extends value {
		speak() {
			super.speak();
			console.log("Hello from decorator");
		}
	};
}

@addVersion("1.0.0")
@Mixin
class Foo {
	speak() {
		console.log("Hello from class");
	}
}

console.log(Foo.version); // "1.0.0"
const foo = new Foo();
foo.speak(); // "Hello from class" "Hello from decorator"
```

There are two ways for decorators to add/modify the class's API surface:
1. Return a new subclass of the original class
2. Use `addInitializer()` to do stuff with the original class

However, these are essentially a nicer, more declarative way to accomplish the same as one can accomplish using existing userland patterns:
1. Return a new subclass of the original class: basically a better way to do [subclass factories](prior-art.md#subclass-factories-mixins).
2. Use `addInitializer()` to do stuff with the original class: basically a better way to do [prototype mutations](prior-art.md#prototype-mutations).

Additionally, while the mixin _application_ is declarative, the mixin logic is still very imperative, making it impossible to disentangle where each part comes from.

## Definitions

In the following, we use these terms:
- **Implementing class**: The class _using_ the mixin.
- **Partials**: A generic concept that is meant to encompass all possible patterns for abstracting class behaviors, including mixins, traits, protocols, etc.

## Non-goals / Out of scope

### Abstract methods

Ultimately, partials need to be able to define both a **contract** (that the implementing class needs to implement) and a **public API** (that the implementing class will incorporate or override).

However, this is not specific to partials: abstract methods are useful for superclasses as well, and partials are independently useful even without abstract methods.
Therefore, this is out of scope for this proposal and can be pursued separately.

### Parameterization syntax

While parameterized partials are incredibly useful,
given the dynamic nature of the language, they can always be implemented as a function that returns a partiall, regardless of the specifics of the syntax.

## Requirements

### Extending API surface of implementing class

Many of the use cases for partials need to extend the API surface of the implementing class (new properties and methods).
While most use cases are around instance fields, extending statics should also be supported.

### Operate on prototypes, not instances

All API surface extension should be done on the class prototype, not individual instances, both for performance reasons and to make introspection easier.

### Adding side effects to existing methods

Besides adding new API surface, partials need to be able to extend existing methods with new behavior.
This is typically needed for APIs that use methods as lifecycle hooks, such as web components.

In nearly all of these cases, the need is to add side effects to existing void methods, not to modify
return values.
For most use cases that actually _do_ need to affect return values, it seems like subclassing (or subclass factories) might be a better fit.

However, it is important to **distinguish intentional composition from unintentional naming collisions**,
so perhaps this should be opt-in, via some kind of syntax or annotation to declare a method as a side effect that can be composed with others of the same name.

These side effects would run in a deterministic order, either before or after the implementing class's method (if one exists).

### It should be possible to apply partials to an existing class

For many use cases, it is _essential_ to be able to extend a class _after_ it has been defined.

One of these use cases is [custom attributes](https://github.com/w3c/tpac2025-breakouts/issues/46), which effectively add partials to existing element classes (typically `HTMLElement`), and this can happen at any point in time.

But even for other use cases, being able to apply partials to an existing class decouples them, and makes it possible to develop them independently, without either having to know about the other.

Designs based on [subclass factories](prior-art.md#subclass-factories-mixins) do not support this, whereas [protocols](#first-class-protocols-proposal) do.

### Static reflection

It SHOULD be possible to test whether a given class implements a partial, possibly via a new operator (`implements`? `uses`?).

### Encapsulation

Partials need to be able to have their own encapsulated state that is not exposed to the implementing class.
This is important for UAs to be able to expose partials that userland code can use without having to expose their internals.

It seems reasonable that partials should not have access to private fields of the implementing class, since they may be developed via entirely separate entities.
This is also the case today for inheritance.
For cooperative partials, authors can always use naming conventions.
If down the line, the language gets `protected` fields, partials could have access to those.

## Nice to haves

### Single declaration of intent

[It’s an API design antipattern](https://lea.verou.me/blog/2025/user-effort/#signal-to-noise) if users should need to specify the same intent more than once.

That’s an issue with the [First-Class Protocols proposal](https://github.com/tc39/proposal-first-class-protocols):
intent is declared twice, once by setting the protocol property (usually a `Symbol`) and once by calling `Protocol.implement()`.

Intent duplication does not just add friction, it also creates an error condition if the two do not match.
It’s easy to set the property and forget to call `Protocol.implement()`, so then testing for the presence of the property is not enough to know if the protocol is implemented.
Worse yet, what happens if you call `Protocol.implement()` _without_ setting the property?

### Instance reflection

If static reflection is possible, then one can test whether a given instance implements a given partial through a utility method:

```js
function getSupers (Class) {
	let classes = [];

	do {
		classes.unshift(Class);
		Class = Object.getPrototypeOf(Class);
	} while (Class && Class !== Function.prototype)

	return classes;
}

function implementsPartial (instance, Partial) {
	return getSupers(instance.constructor).some(Class => Class implements Partial);
}
```

That said, it would be _nice_ if authors don’t have to write this code themselves,
and can simply use an existing method or operator to test whether a given instance belongs to a class that implements a partial (or implements a partial itself — depending on how we define it this may or may not be possible).

In some designs, repurposing `instanceof` to work for partials could be reasonable.

### Use existing class primitives

There is a lot of existing machinery in the language around classes and inheritance that would also be useful for partials.

E.g.
- Partials could have their own inheritance chain, so that more specific/complex partials can be built on top of simpler/less opinionated ones.
- Partials also benefit from features like private class fields, decorators, etc.
- Since partials need to be able to add both instance and static fields, declaring them using the same syntax makes sense.

Additionally there is a DX argument here too: given that the final product of a partial is a class, the closer the partial definition is to that class, the easier it is to write (see [natural mapping](https://en.wikipedia.org/wiki/Natural_mapping_(interface_design))).

### Reversibility

Since partials can be added after the fact, it would be nice if they could be removed as well.
This would allow dynamically applying them based on some condition, and later removing them when a condition is no longer met.

### Declarative class syntax

While being able to apply partials to an existing class is a core requirement, there are many cases where partials are applied as mixins during class definition time, and it would be good to have a nice declarative syntax for it that lets authors keep everything in one place.

## Design space

Instead of enumerating individual proposals, it may be useful to first explore the different design decisions separately.
These fundamental decisions can be combined in different ways to produce a much larger number of proposals.

### 1. Are partials syntactically distinct from classes?

It seems clear that partials should follow a similar syntax to classes, but how similar?
Are partials just classes that can be repurposed, or syntactically distinct constructs that share a lot with classes?

On one side of the spectrum, partials could even be defined as **regular classes with their own inheritance chain** that can even be used independently.
For example,

```js
class HasIcon extends HTMLElement { /* elided */ }
class MyButton extends HTMLButtonElement with HasIcon { /* elided */ }
```

Pros:
- The partial doesn't even need to know it’s a partial — any class can be repurposed as a partial.
- All present and future class primitives just work.
- Implementing class can decide between inheritance or composition depending on intent.
- Less new syntax needed.
- More flexible, since partials can have partial-specific syntax which is not allowed in regular classes, and there can be class syntax that is not allowed in partials.

On the other side, partials are syntactically distinct from classes, defined using a different keyword, e.g. `partial`, `mixin`, `trait`, etc.:

```js
partial HasIcon { /* elided */ }
class MyButton extends HTMLButtonElement with HasIcon {}
```

This is the direction most languages seem to have gone with when they have an actual concept of partials.
Languages where partials are implemented as classes are typically framed as multiple inheritance languages (e.g. [Python](prior-art.md#python-multiple-inheritance)).

This permits more syntactic flexibility: partials can have partial-specific syntax which is not allowed in regular classes, and there can be class syntax that is not allowed in partials.
For example:
- A different keyword for the partial’s own state and inheritance chain vs the eventual object instance
- Keywords to define how fields and methods are combined with those of the implementing class and/or other partials (e.g. `compose`, `override`, `before`, `after` etc).

There is also the possibility of hybrids:
- An interesting pattern is seen in [Dart](prior-art.md#dart-mixins), which supports both `mixin` and `mixin class` as distinct concepts.
- Perhaps partials _are_ classes, but classes are not partials.
- Perhaps both partials and classes can be applied to other classes as partials, but any partial-specific features require defining a partial.
- Perhaps all classes can have annotations around how their methods are composed with another class, and when used as standalone these annotations are simply ignored.

### 2. How and where is the partial included?

The major options are:
1. Include in the class prelude (e.g. `class A extends B with Partial {}`)
2. Include in the class body (e.g. `class A extends B { use Partial; }` or even a class version of spread syntax: `class A extends B { ...Partial }`)
3. Entirely separate declaration (e.g. `implement Partial for A {}`), like [Rust traits](prior-art.md#rust-traits)

Pros & Cons:
- A syntax that goes in the class body could provide more flexibility wrt how overrides are handled, but we lose the ability to see at a glance what a class is made of.
That said, it could be useful for macro-like mixins, which function essentially as shorthands for specifying the fields verbatim.

Since a core requirement is the ability to apply partials to an existing class, it seems that a syntax along the lines of 3 would be useful _anyway_, but would be awkward if it were the only way to apply partials.
However, if we have 3, then we can also have 1 as syntactic sugar for partials applied at class definition time.

Additionally, while 1 and 2 may appear equivalent at first glance, they sit at slightly different levels of abstraction, with 2 feeling lower-level than 1.

### 3. Is composition distinct from inheritance?

There are two main options:
1. Partials are a separate concept, overlaid on top of the inheritance chain.
2. The partial is a regular class that is included in the inheritance chain.

Note that what `super` resolves to is somewhat orthogonal:
- `super` skips partials and refers to the superclass. There may or may not be syntax to access the partial's state from the implementing class (e.g. `partial.super`). See [Java 8+ interfaces](prior-art.md#java-8-interfaces) for an example of this.
- `super` resolves to the last applied partial. This is how [subclass factories](prior-art.md#subclass-factories-mixins) work, but there are languages that follow this pattern even when their partials do not affect the inheritance chain, because the way they resolve it is method-specific.

In most languages the inheritance chain is a separate concept and is unaffected by partials.

The languages where partials affect `super`are:
- [Python](prior-art.md#python-multiple-inheritance) (multiple inheritance, no actual partials)
- [Scala mixins](prior-art.md#scala-mixins) (`super` resolves in a method-specific way)
- [Ruby mixins](prior-art.md#ruby-mixins) (`super` resolves to the same _method_ up the chain)

Pros of piggybacking on inheritance:
- Existing language primitives just work. Execution order, conflict resolution, etc. are all well defined.
- `instanceof` could be made to just work for testing whether a class implements a given partial (depending on how the feature is designed — e.g. with subclass factories it doesn’t currently work).

Cons:
- Conflates behavior and identity, which is exactly what we're trying to avoid. The inheritance chain is polluted and no longer makes sense to traverse.
- Feels more obtrusive, since even including a rather minor utility has a major effect in the inheritance chain.
- Easy to accidentally shadow in naming conflicts.
- Slight compat risk when adopting a new mixin, in case any class consumers were depending on the inheritance chain

### 4. Do partials operate on their own state or the full composed instance?

This is intrinsically linked to the previous question.
For many cases, this distinction does not matter.
But suppose you have two partials, each defining an `init()` method, which is called at a certain point to perform some initialization tasks and needs to be called on both the current instance and its superclass:

```js
class M1 {
	init() {
		super.init();
		// ...
	}
}

class M2 {
	init() {
		super.init();
		// ...
	}
}

class A extends B with M1, M2 {
	// ...
}
```

When each mixin calls `super.init()`, the intent is to initialize its own state on the parent instance, not to also initialize behavior by any other mixin which happens to define the same method.

Admittedly, for this particular example, since `init()` is not really public API, the problem can be circumvented by making it private or using a `Symbol`.
However, there are other cases where the method _is_ part of the public API, so this is not a workable solution.

Worse yet, with the inheritance chain model, what happens with each `super.init()` call is subject to the order of inclusion, which is not always meaningful.

### 5. Can the partial's state be accessed from the implementing class?

Just like with inheritance it is often desirable to access the parent state via `super`,
the same need exists for mixins.

In [Java 8+ interfaces](prior-art.md#java-8-interfaces), this is achieved via `InterfaceName.super`, while plain `super` is reserved to follow the inheritance chain.

### 6. How to handle naming conflicts?

Even if authors are encouraged to use symbol names for anything that does not need to be part of the public API, and even if composable side effects are explicitly annotated as such, there is always the risk of naming conflicts between public fields that simply happen to use the same name.

One solution is to simply **throw** an error.
This makes the issues more discoverable, but also makes the logic more fragile.

Arbitrarily selecting a default precedence order (e.g. first or last one wins) or defaulting to some form of composition could lead to strange bugs, because in many cases these methods may be intended to accomplish entirely different things, and the naming collision is incidental.

A way for the implementing class to rename certain fields when it is aware of the conflict would help alleviate the problem, but does not eliminate it entirely, because unforeseen conflicts still need to be handled in a reasonable way.

In designs where overrides are handled via a last-one-wins mechanism, it can be very useful to have a means to access any overridden method (akin to `super` — or even `super` itself).

Perhaps overrides could only be allowed via an explicit opt-in (e.g. an `override` keyword), to eliminate accidental naming collisions.

There is also the question of _what_ is a naming collision.
E.g. if a superclass implements a method but the implementing class does not, is that a naming collision?
There are many use cases where a partial is pulled in to implement a method "properly" which has a stub implementation in some superclass (e.g. `toString()`, `toJSON()` etc.).

### 7. Are partials inherited?

Depending on the design, partials may or may not be inherited by subclasses.
What is most useful?
It seems that for most use cases, inheritance is desirable.
Does that generalize?

## Concrete ideas (strawmans)

There are two components to addressing the requirements with a concrete proposal:
1. Function side effects: How to extend the methods in the implementing class with new behavior?
2. Partial syntax: How to declare and apply partials to a class?

### Function side effects

Many use cases require the ability to mix in partials to an existing class, which is why it’s listed as a core requirement above.
[Protocols](#first-class-protocols-proposal) allow post-hoc extension, but treat all naming conflicts as errors.
However, for many use cases the ability to add side effects to existing methods is crucial.
This includes all cases where functions are used as lifecycle hooks, such as all the web components use cases.
While it could be argued that ideally, a pub/sub mechanism would be more suitable and naturally composable, the reality remains that this is a widely used pattern.

[Subclass factories](prior-art.md#subclass-factories-mixins) support extending methods via inheritance, but this is not in-place and requires all side effects to be declared at class definition time.

Therefore, we need some way to add side effects to existing functions (both methods and accessors), either in-place (preserving references) or by creating a new function which can be extended with side effects.

Neither of these needs to be specific to partials, depending on the design, they can be implemented as broader features of `Function` objects.

Below is a detailed exploration of the design space for function side effects, from the least controversial to the most controversial design decisions.

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

Another open question is around **reflection**.
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

#### In-place vs immutability-preserving

That is probably the hairiest part of the design space.
The ability to add side effects in place, to any function, without breaking references to it can be incredibly powerful for a number of use cases extending beyond class partials and allows for class partials that are minimally invasive.

It does break assumptions around immutability, but per the [Priority of Constituencies](https://www.w3.org/TR/design-principles/#priority-of-constituencies), philosophical purity is secondary to author needs.

But if in-place side effects are too controversial, another idea that preserves immutability is to restrict the ability to have a mutable list of side effects to a **special function type**.
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

### Syntax ideas for class partials

TBD
