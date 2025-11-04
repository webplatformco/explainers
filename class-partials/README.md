# Class Partials

Authors: Lea Verou

> [!IMPORTANT]
> This is a work in progress. Feel free to send PRs to improve it, but this is not the time to evaluate it critically.

<details open>
<summary>Contents</summary>

1. [Use cases](#use-cases)
	1. [Use case axes/dimensions](#use-case-axesdimensions)
	2. [Concrete examples](#concrete-examples)
	3. [API surface for iterables](#api-surface-for-iterables)
	4. [`HTMLMediaElement`](#htmlmediaelement)
	5. [Custom Elements](#custom-elements)
	6. [Custom Attributes](#custom-attributes)
2. [Prior art](#prior-art)
	1. [Userland patterns](#userland-patterns)
	2. [Other languages](#other-languages)
3. [Resources](#resources)
4. [Existing proposals](#existing-proposals)
	1. [Mixins proposal](#mixins-proposal)
	2. [First-Class Protocols proposal](#first-class-protocols-proposal)
	3. [Decorators proposal](#decorators-proposal)
	4. [Older proposals](#older-proposals)
5. [Definitions](#definitions)
6. [Non-goals / Out of scope](#non-goals--out-of-scope)
	1. [Abstract methods](#abstract-methods)
	2. [Parameterization syntax](#parameterization-syntax)
7. [Requirements](#requirements)
	1. [Priority of constituencies](#priority-of-constituencies)
	2. [Extending API surface of implementing class](#extending-api-surface-of-implementing-class)
	3. [Operate on prototypes, not instances](#operate-on-prototypes-not-instances)
	4. [It should be possible to apply partials in-place](#it-should-be-possible-to-apply-partials-in-place)
	5. [Function composition](#function-composition)
	6. [Static introspection](#static-introspection)
	7. [Encapsulation](#encapsulation)
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


</details>

This document is currently an exploration of the problem and design space, and does not yet propose any specific solution.

> [!NOTE]
> To avoid assumptions about specific patterns, this document avoids using terms like _mixins_, _traits_, _protocols_, etc. and instead uses the more general term _partials_ which is meant to encompass all of them.

## Use cases

### Use case axes/dimensions

The limitations of single inheritance are well-established in the literature, and there is little point in revisiting them here.

On a high level, there are two distinct axes of use cases with different requirements:
1. **Level of coupling** between the behavior, the class, and the code applying the behavior to the class. This ranges from the same entity developing all three and seeking to simply reduce knowledge duplication (**cooperative partials**), to completely decoupled development where all three are developed independently by different entities (**decoupled traits**).
2. **Abstractness**: Does the partial require any API surface from the implementing class (essentially as input to parameterize it), or does it provide all the API surface it needs?
3. **Type**: Is the partial a concrete behavior (_has a_) that can be reasoned about as a separate concept, or an aspect of the class's identity (_is a_) that we simply abstracted away for maintainability?

Depending on where a use case falls on these axes, the requirements and constraints will be different.

For example, having a way to apply class partials to existing classes is much more important for decoupled traits than for cooperative partials.
And solutions that involve modifying the constructor or inheritance chain may be more acceptable for identity partials than for behaviors/traits.

### Concrete examples

> [!NOTE]
> Issue: This section is currently too Web focused. We need more pure ES examples, and/or examples from JS runtimes.

### API surface for iterables

Because the language has no primitive for interfaces, iterables are implemented as a protocol: A `Symbol.iterator` (or `Symbol.asyncIterator` for async iterables) property on the class prototype that returns an iterator object, which is simply an object that needs to implement certain methods (e.g. `next()`, `return()`, `throw()`).

Because iterators are implemented as a protocol and do not add any new API surface to the host class, actual API surface needs to be added separately.
For example, there are many `Array` methods that are useful for all iterables, such as `forEach()`, `map()`, `filter()`, `reduce()`, etc.
But they are inconsistently supported on other iterables as they need to be advocated, specced, and implemented as separate features, rather than an automatic consequence of an object being iterable.

#### `EventTarget`

The `EventTarget` class begun as a DOM API, but has now practically become the web platform's de facto pub/sub mechanism for classes, and has even been adopted by JS runtimes.
Some of its direct subclasses in the web platform include `Node`, `Window`, `IDBRequest`, `AudioNode`, and `AudioContext`.

This has both practical and philosophical issues:

Philosophically, being able to receive events is a capability, not a part of an object’s identity.
You would not describe the `Node` class as "an event target".
`Node` and `AudioContext` have nothing in common, yet they have the same base class.
This is because inheritance is used to apply partials, because the language has no other mechanism for doing so.

Practically, this makes it impossible to have an event target class that also extends another class that does not extend `EventTarget`.
As a trivial example, suppose you had an `ArrayStream` class which was implemented as either an `Array` subclass or a `ReadableStream` subclass and allowed you to read streamed remote data as an array.
You cannot make it an event target (e.g. for events around fetching progress) because neither `Array` nor `ReadableStream` extend `EventTarget`.

In terms of the three axes, `EventTarget` is:
1. **Level of coupling**: Decoupled. `EventTarget` is developed by different entities than the classes that extend it.
2. **Abstractness**: Concrete. `EventTarget` adds API surface, and does not mandate any particular contract to be followed by the implementing class.
3. **Type**: Behavior. `EventTarget` is a concrete behavior that can be reasoned about as a separate concept, not part of the class's identity.

### `HTMLMediaElement`

The `HTMLMediaElement` class is the base class that both `<audio>` and `<video>` elements extend from, and it adds a ton of API surface for controlling media playback.
In terms of the three axes, `HTMLMediaElement` is:
1. **Level of coupling**: Coupled. `HTMLMediaElement` is generally developed by the same entity as the classes that extend it.
2. **Abstractness**: Concrete. `HTMLMediaElement` adds all the API surface it needs.
3. **Type**: Behavior. `HTMLMediaElement` is a concrete behavior that can be reasoned about as a separate concept, not part of the class's identity.
While it can be argued that being a media element _is_ part of the class's identity ()

### Custom Elements

Custom Elements is a Web Components API that allows authors to define new HTML elements by defining a subclass of `HTMLElement` and registering it with the browser.

```js
class MyElement extends HTMLElement {
	// ...
}

customElements.define("my-element", MyElement);
```

> [!NOTE]
> Only extension of `HTMLElement` is currently allowed, no other `Element` subclasses (e.g. `SVGElement`) can be extended, and no `HTMLElement` subclasses can be extended either (e.g. `class MyButton extends HTMLButtonElement` is not allowed).

Since Custom Elements is a fairly low-level API, and the space of UI components is vast,
the need for partials is very strong in this area.
Some examples:
- Form associated: manage API surface and behaviors of custom elements that are also form controls
- Styles: Read a predefined static property (e.g. `styles`) and take care of fetching and applying adopted stylesheets
- Props: Read a predefined static property (e.g. `props`) and take care of setting up attribute-property reflection

Since use cases are so diverse, they also span the entire spectrum of the three axes:
Some are coupled, some are decoupled, some are concrete, others include abstract parts, some are behaviors, others are parts of a component's identity.

The lack of a mechanism for class partials has led to implementing [controller](prior-art.md#controllers)-based solutions for current needs, through `ElementInternals`, which requires a lot of glue code by the web component author.
For example, making a custom element that is also a form control looks like this:
```js
class MyElement extends HTMLElement {
	// This tells ElementInternals that this element is form associated
	static formAssociated = true;

	constructor() {
		super();

		// Cannot use #internals because subclasses need access
		this._internals = this.attachInternals();

		this.addEventListener("input", () => {
			this._internals.setFormValue(this.value);
		});
	}

	// API glue code
	get labels () { return this._internals.labels; }
	get form () { return this._internals.form; }
	get validity () { return this._internals.validity; }
	get validationMessage () { return this._internals.validationMessage; }
	willValidate (...args) { return this._internals.willValidate(...args); }
	reportValidity(...args) { return this._internals.reportValidity(...args); }
	checkValidity(...args) { return this._internals.checkValidity(...args); }
	// ...
}
```

Not only is this tedious and error-prone, it also means that when there is new API surface for form controls, authors must manually update their glue code to support it.

More recently, similar solutions are being proposed even for identity-based partials (see  [`ElementInternals.type`](https://github.com/whatwg/html/issues/11061)) so the need for a language-level solution is even more pressing.

### Custom Attributes

Just like inheritance is not suitable for all class logic sharing, in UIs components are not suitable for all UI reuse.
Some functionality is fundamentally a trait that should be possible to apply to any element, rather than element identity that should be applied as a dedicated element type.

Many HTML global attributes are such traits:
- `title` adds a tooltip to any element. A `<title>` component would have been much more limiting.
- `hidden` hides any element. A `<hidden>` component would have been much more limiting.
- `popover` makes any element a popover. While a dedicated `<popup>` or `<popover>` component was discussed, it was decided that a global attribute was much more flexible.

While there is no current API to create custom attributes, there are several proposals to do so, and it seems like a better path forwards for many use cases around extending built-ins.

Custom attributes are a great example for why there should be a mechanism to apply partials to existing classes.
It’s not merely a convenience — solutions that require generating a new constructor to add a partial are a no-go for custom attributes, since the constructors are invoked by writing HTML, not by directly calling `new`.

While there could be a new API for adding custom attributes that manually manages lifecycle hooks, a language-level solution would both reduce the need for new API surface and provide more flexibility.


## Prior art

### Userland patterns

See [Prior art](prior-art.md#userland-patterns) for a detailed exploration of the existing userland patterns:
1. [Subclass factories (mixins)](prior-art.md#subclass-factories-mixins)
2. [Controllers](prior-art.md#controllers)
3. [Prototype mutations](prior-art.md#prototype-mutations)
4. [Instance mutations](prior-art.md#instance-mutations)

### Other languages

See [Prior art: Other languages](prior-art.md#other-languages) for a detailed exploration of the primitives other languages have implemented to address similar use cases.

## Resources

- [Wikipedia: Multiple inheritance](https://en.wikipedia.org/wiki/Multiple_inheritance)
- [Wikipedia: Virtual inheritance](https://en.wikipedia.org/wiki/Virtual_inheritance)
- [Wikipedia: Mixins](https://en.wikipedia.org/wiki/Mixin)
- [Wikipedia: Traits](https://en.wikipedia.org/wiki/Trait_(computer_programming))
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

### Older proposals

- [Strawman: Trait composition for Classes](https://web.archive.org/web/20160313141617/http://wiki.ecmascript.org/doku.php?id=strawman:trait_composition_for_classes)

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

### Priority of constituencies

As a generalization of the web platform's [priority of constituencies](https://www.w3.org/TR/design-principles/#priority-of-constituencies) based on the idea of [_consumers_ over _producers_](https://lea.verou.me/blog/2025/user-effort/#consumers-over-producers), the needs of the implementing class author should take precedence over the needs of the mixin/trait/partial author, which take precedence of the needs of the underlying platform.

### Extending API surface of implementing class

Many of the use cases for partials need to extend the API surface of the implementing class (new properties and methods).
While most use cases are around instance fields, extending statics should also be supported.

### Operate on prototypes, not instances

All API surface extension should be done on the class prototype, not individual instances, both for performance reasons and to facilitate introspection.

### It should be possible to apply partials in-place

For many use cases, it is _essential_ to be able to extend a class _after_ it has been defined.

It is not always practical to replace classes with new ones, e.g. because instances have already been created and/or are not created via `new` but some other mechanism (e.g. HTML parsing, literals, etc.).

But even for other use cases, being able to apply partials to an existing class decouples them, and makes it possible to develop them independently, without either having to know about the other.

No design based on [subclassing](prior-art.md#subclass-factories-mixins) can support this.
Out of existing proposals, only [protocols](#first-class-protocols-proposal) allow in-place application.

### Function composition

In many designs, naming conflicts are treated either as errors, or via precedence rules about what overrides what.
However, opt-in _composition_ can enable more flexible and powerful patterns.
Besides adding new API surface, partials often need to be able to seamlessly extend existing methods with new behavior.

[Subclass factories](../prior-art.md#subclass-factories-mixins) sidestep function composition by piggybacking on inheritance which has established, explicit mechanisms for this.
But any non-inheritance-based design needs a way to compose functions.

On a high level, the options are:
1. **Error**: Treat all naming conflicts as errors.
This is what [Protocols](../prior-art.md#first-class-protocols-proposal) do.
1. **Function precedence**: Pick the function to execute based on some precedence algorithm, ignore all others. This is how many languages handle naming conflicts between class partials.
3. **Return value precedence**: Execute all functions, pick the return value based on some algorithm.

Treating all naming conflicts as errors is a no-go, as it should be possible to develop partials independently, without having to be aware of the API of every other partial.

Additionally, beyond accidental collisions there is also often **composition intent**, where a partial is fully aware that the implementing class may have a method with the same name, and wants to compose with it.
A common reason for this is when **functions are used as lifecycle hooks**.
For example, in Web Components, authors react to component lifecycle events by running code on methods like `connectedCallback()`, `disconnectedCallback()`, `attributeChangedCallback()`, etc.
While it could be argued that ideally, a pub/sub mechanism would be more suitable and naturally composable for these use cases, this doesn't change the fact that it is a widely used pattern.
It could even be argued that effectively, OOP inheritance is also an expression of this pattern: authors can schedule initialization logic to run at instance creation time by adding it to a method with a particular name (`constructor()`).
Arguably, even `toJSON()`, `valueOf()`, `toString()`, etc. are also expressions of this pattern.

It could be argued that composition intent is known at function definition time, and thus could be explicitly declared via some kind of function annotation (e.g. `composable foo()`).
However this is orthogonal: regardless of _which_ functions are composed, the question of _how_ to compose them remains open.

If lifecycle hooks are the main use case, perhaps function composition can be restricted to side effects, and subclassing could still be the recommended mechanism for overriding return values.

Function composition is explored in depth in the [Mutable Functions](proposals/mutable-functions.md) proposal.

### Static introspection

It should be possible to test whether a given class implements a partial without creating an instance.
Some ways this could be done:
- A new operator (`implements`? `uses`?).
- An iterable of all implemented partials, e.g. `Class[Symbol.partials]`
  - This could even be writable to _add_ a new partial.

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

Proposals are developed as separate files in the [proposals](proposals) directory.
