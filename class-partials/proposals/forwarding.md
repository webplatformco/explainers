# `Object.forward()`

> [!NOTE]
> This is a work in progress and not yet ready for review.

## Motivation

There are many use cases where one wants to forward property access and method calls to a [delegate object](https://en.wikipedia.org/wiki/Delegation_pattern).
Accessors can do this, but need to be manually authored for each property.
Proxies can do it in aggregate, but are often too heavyweight for the use case, and return a new object reference rather than mutating the original object.

### Use cases

It should be possible to apply this to a class and its prototype, to facilitate the [delegation pattern](https://en.wikipedia.org/wiki/Delegation_pattern).

```js
class Delegate {
	foo() {  /* ... */ }
	bar() {  /* ... */ }
	qux = 1;
}

class MyClass {
	constructor() {
		this.delegate = new Delegate();
	}

	foo() {
		return this.delegate.foo();
	}

	bar() {
		return this.delegate.bar();
	}

	get qux() {
		return this.delegate.qux;
	}
	set qux(value) {
		this.delegate.qux = value;
	}
}
```

That means that a naive signature like this:

```js
Object.forward(target, source);
```

…would not work, since the target object is different per instance.
We could of course do `Object.forward(this, this.delegate)`, in the constructor, but ideally we want to be able to get the same performance as manually authoring the accessors.

This means we need to be able to _read_ the property names statically even if the actual delegation target depends on the instance.
Which means we need to be able to specify the property names separately from the delegation target.

Perhaps something like:

```dts
Object.forward(target: Object, source: Object, {
	getTarget: function,
	properties: Iterable<string | symbol>,
});
```

## Design space

TBD

## Proposal

TBD
