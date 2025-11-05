# Composition via delegation / forwarding

> [!NOTE]
> This is a work in progress and not yet ready for review.

A common OOP pattern is to achieve composition via delegation

## Motivation

Delegation is a userland pattern for sharing functionality between classes, described in [Prior art](prior-art.md#delegation).

The fact that [Controllers](prior-art.md#delegation) are entirely separate and can even have their own inheritance chain is very appealing and can be great for composition.

However, without any new language primitives, there are some severe drawbacks:
- No way to add API surface to the class, so use cases that need it involve a lot of repetitive glue code
- Because the glue code is manually authored by the implementing class author, it may not be up to date with the latest API surface of the controller.
- No good way to extend existing methods in the host class, e.g. to add side effects to certain lifecycle hooks (at least not out of the box).
- No way to check whether a given class implements a certain controller or not (without a shared contract about what property the controller is stored in)
- No way to add a controller to a class from the outside, the implementing class needs to create the controller itself.

What if we could introduce language primitives that eliminate these drawbacks?
In fact, some of the existing proposed primitives already address a lot of these issues:
- Some way to automatically generate glue code (either for all public fields or a subset). Perhaps even [class spread syntax](class-spread.md) + a userland helper might be enough for this.
- A way to inspect a class for its delegates, without creating an instance.
But this is an instance of the more general problem of [inspecting public class fields](customizable-fields.md).
- In fact, with [customizable public class fields](customizable-fields.md), delegates can be added post-hoc to an existing class through third-party code.

## Considerations

### How much API surface to forward?

- In some cases, the desire is to transparently expose all of the delegate's public API on the implementing class, whereas in other cases, only a subset (either a whitelist or a blacklist of fields).
- In some cases the desire is to expose the API with the same names as the controller, whereas in other cases, the names are transformed.
- Ultimately, all properties can be exposed as accessors:
 - Methods can just be exposed as accessors returning the method from the delegate
 - Regular class fields should also be exposed as accessors
 - And obviously, accessors should be forwarded too.

```js
class Controller {
  foo = 1;
  bar() { /* ... */ }
  get baz() { /* ... */ }
  set baz(value) { /* ... */ }
}

class Foo {
  constructor() {
    this.controller = new Controller();
  }

  get foo() { return this.controller.foo; }
  set foo(value) { this.controller.foo = value; }

  get bar() {
    return this.controller.bar;
  }

  get baz() {
    return this.controller.baz;
  }

  set baz(value) {
    this.controller.baz = value;
  }
}
```

But is this really specific to controllers, or even classes?
It seems like a cross between a Proxy and accessors, where we want accessor semantics but Proxy convenience.
This seems like a more broadly useful primitive.

## Design space



If at least one of the two is made declarative, then it would be enough to generate the syntactic sugar.

E.g. one potential direction could be to use regular classes for controllers and declare them as controllers at the implementing class so their API can be added to the implementing class:

```js
class FooController {
	constructor(instance, options = {}) {
		this.host = instance;
	}

	foo () { /* elided */ }
	get bar() { /* elided */ }
	set bar(value) { /* elided */ }
}

// Commented out code is generated automatically
class Foo {
	// maps to `this.controller = new FooController(this);
	use controller = FooController;
	// maps to `this.controller = new FooController(this, options);`
	use controller = FooController(options);

	constructor() {
		// Generated automatically:
		this.controller = new Controller(this);
	}

	/* Generated automatically:
	foo () {
	 return this.controller.foo();
	}

	get bar() {
		return this.controller.bar;
	}

	set bar(value) {
		this.controller.bar = value;
	}
	*/
}
```

By default, the adopted API would include all own public fields, but it should be customizable.

## Does the relationship need to be bidirectional?

If the main value of first-class controllers is generating glue code for the API surface, then there is no need for a bidirectional relationship.
All we need is for the implementing class to have a reference to the controller.
Whether the controller has a reference to the implementing class is irrelevant.

This means controllers can extend to addressing patterns where you want to reuse behavior _without_ the class being reused actually having the same identity as the implementing class.


Another benefit of this pattern is that it facilitates the use cases where you _don't_ actually want to extend a given class, but you want to adopt methods from it that operate on an instance.

E.g. suppose you wanted to have an `<a-button>` web component that behaves as both a button and a link, a very common use case.
Assuming DOM APIs actually behaved as regular classes, one could simply do:

```js
class AnchorButton extends HTMLButtonElement {
	use anchor = new HTMLAnchorElement();
}
```

Which would generate glue code for a ton of `HTMLAnchorElement` fields (`target`, `download`, `ping`, `rel`, `relList`, `hreflang`, `type`, `referrerPolicy`, `text`, `coords`, `charset`, `name`, `rev`, `shape`, `origin`, `protocol`, `username`, `password`, `host`, `hostname`, `port`, `pathname`, `search`, `hash`, `href`, `interestForElement`, `constructor`, `hrefTranslate`)
