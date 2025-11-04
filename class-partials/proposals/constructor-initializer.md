# Instance initializers / Constructor side effects / Customizable `[[Construct]]`

> [!NOTE]
> This is a work in progress and not yet ready for review.

## Motivation

### In-place class partials

Currently, there is no way to add side effects to an existing class constructor without subclassing.
While this is a more general problem with [functions being immutable](mutable-functions.md),
it is particularly acute for constructors because there is no way to replace a reference to a class constructor without replacing a reference to the class itself.

While it may be acceptable to replace references to methods when applying partials,
e.g. through wrapping with a [mutable function](mutable-functions.md),
that does not work for constructors because replacing the constructor means replacing the class itself, defeating the entire purpose of in-place partials.

However, this is a problem that extends beyond class partials.

### A way for superclass to run logic after the instance has been fully constructed

Superclasses often want to run logic after the instance has been fully constructed, including any subclasses.
The common pattern is to define a separate method for this purpose, and force all subclasses to call it.

```js
class Super {
	postConstruct () {
		console.log("Super postConstruct");
	}
}

class Sub extends Super {
	constructor () {
		super();
		// own logic here...
		this.postConstruct();
	}
}
```

However, if we see subclassing as using an API, this puts the burden on consumers, which is suboptimal and error-prone. It should be possible for the superclass to handle this by itself, without requiring boilerplate from subclasses.

Currently, the only way to do that is by exploiting microtasks:

```js
class Super {
	constructor () {
		Promise.resolve().then(() => {
			// Run logic after the instance has been fully constructed
		});
	}
}
```

Not only is this hacky and obscures user intent, it is also async, and thus does not run immediately after instance creation.

### Initialization logic on prototype assignment

Consider this:

```js
class A { constructor () { console.log("A constructor") } }
class B extends A { constructor () { super(); console.log("B constructor") } }
let a = new A(); // "A constructor"
Object.setPrototypeOf(a, B.prototype); // nothing happens
```

While it could be argued that in most cases, there is a better way than `Object.setPrototypeOf()`,
there are also cases where there is no other solution.

Additionally, there are other non-constructor ways to create instances:

```js
let a = Object.create(A.prototype); // nothing happens
a instanceof A; // true
```

## Proposal

If authors could set a method that runs initialization logic for every new instance, regardless of how it was created, it would solve all of these problems in one go, and provides a method that can be wrapped and/or replaced without affecting references to the class itself, which is very useful for class partials.

#### Too powerful?

Given how prototypal inheritance works, introducing a method that allows tracking object creation across realms, for all classes (including built-ins), can be _very_ powerful.

It basically allows tracking object creation regardless of where or how the object was created.
This includes literals like `function() {}`, `[]`, `/regex/` but also HTML parsing into DOM nodes.

E.g. one could track every single `HTMLElement` created in a page (regardless of shadow root) and perform changes to it:

```js
HTMLElement.prototype[Symbol.initialize] = function () {
	if (this.hasAttribute("title")) {
		// Replace with fancy tooltip component
	}
}
```

This raises several concerns:
1. Does this break encapsulation too much? Could it leak implementation details of the JS engine?
2. Is it not possible to implement something as broad performantly?
3. Does this violate the Web's same origin policy? Could a malicious script make assumptions about what cross-origin code is doing based on what objects are being created?

If the answer to any of the above is "Yes" (and I suspect it will be) how can we tweak the design to make it less powerful while still addressing the motivating use cases?

One observation is that it seems that all motivating use cases involve author classes, whereas most of the concerns are around built-in classes.
This is a convenient separation:
perhaps **restricting it to author classes** could be a way forwards that gives us a lot of the value, without the risks.
Ideally, the design should allow extending it to certain "safe" built-ins in the future as needed.

For an alternative solution that is also restricted to author classes, see [class field introspection](customizable-fields.md).

### Design space

#### Sync or async?

Sync is preferable, as it is strictly more powerful: async execution can always be triggered through sync code, whereas the opposite is not true.

However, it's entirely possible that the feature is _too_ powerful to be implemented in a sync way,
so async could be a viable compromise.

#### String or symbol name?

```js
class A {
	initialize () {
		console.log("A initialize");
	}
}
```

```js
class A {
	[Symbol.initialize] () {
		console.log("A initialize");
	}
}
```

A string name has better DX, but any reasonable name will carry huge compat risks.

Veredict: **Symbol**.

#### Initializer or customizable `[[Construct]]`?

One possible framing is around customizing the `[[Construct]]` internal method, via a new known symbol: `Symbol.construct`.

Then, to add side effects to a constructor, one could do:

```js
let nativeConstruct = Class[Symbol.construct];
Class[Symbol.construct] = function (...args) {
	let result = nativeConstruct.apply(this, args);
	// Custom code here...
	return result;
}
```

Then, built-ins that don’t want to support this kind of mutation could simply define the property as not writable and not configurable.

One downside of this approach is that while it addresses the motivating use cases around constructor side effects, it does not address the use cases around instance initializers (even when created in ways that do not invoke the constructor).

#### Name bikeshedding

This explainer uses `initialize`. Other potential names include:
- `construct` (see above)
- `created`
- `postConstruct`
- `prototypeAssigned`
- `onPrototype`

#### Does it require `super[Symbol.initialize]()` to run on superclasses?

Ideally, it should not require `super[Symbol.initialize]()` to run on superclasses, which would more strongly preserve the guarantee that the initializer is always executed.
But is that too much magic?

The simplest mental model is that the initializer is an automatically added `this[Symbol.initialize]?.()` call that is called by the constructor.
However, that would require initializers to manually call `super[Symbol.initialize]()` if they want to run on superclasses.

#### Order of execution

If we only have one class, things are simple: if the constructor is executed, then the initializer should be executed after it.

But how do initializers fit into inheritance?
1. Are they magically executed for superclasses or do authors need to manually call `super[Symbol.initialize]()`?
2. Are superclass initializers executed before or after the subclass constructor?

#### Can it run multiple times for the same object?

Consider this:

```js
class A {
	[Symbol.initialize] () {
		console.log("A constructor");
	}
}

let o = {};
Object.setPrototypeOf(o, A.prototype); // "A constructor"
Object.setPrototypeOf(o, Object.prototype);
Object.setPrototypeOf(o, A.prototype); // Does it run again?
```

#### Does it run on existing instances?

Consider this:

```js
class A {}

let a = new A();

A.prototype[Symbol.initialize] = function () {
	console.log("A constructor");
}
```

What should happen?
If the mental model is that this works a bit like an event that tracks prototype assignment, then it should not run after the prototype has been assigned.
But if the mental model is that this is _guaranteed_ to run on instances of `A` no matter what, then it should run again.
