# Class Spread Syntax

## Motivation

Today, the spread syntax can be used to compose an object from multiple other objects of the same or similar type.
We have it for iterables:

```js
const arr = [...arr1, ...arr2];
```

And for objects:

```js
const obj = { ...obj1, ...obj2 };
```

But we don't have it for classes.

This makes it hard to abstract class behavior out into separate modules,
even in scenarios where class definitions can be cooperatively developed.

## Proposal

The spread syntax for classes would work like this:

```js
class A {
	foo() { console.log("foo"); }
	static bar = "bar";
}

class B {
	...A;
}

console.log(B.bar); // "bar"
(new B).foo(); // "foo"
```

The semantics are largely those of assignment.
Class fields and static initialization blocks aside, it would desugar to something like:

```js
const instanceDescriptors = Object.getOwnPropertyDescriptors(A.prototype);
for (const key in instanceDescriptors) {
	if (key === "constructor") continue;
	Object.defineProperty(B.prototype, key, instanceDescriptors[key]);
}
const staticDescriptors = Object.getOwnPropertyDescriptors(A);
for (const key in staticDescriptors) {
	if (["length", "name", "prototype"].includes(key)) continue;
	Object.defineProperty(B, key, staticDescriptors[key]);
}
```

Unlike [subclass factories](../prior-art.md#subclass-factories-mixins), the spread operator would not affect the inheritance chain, it is essentially a macro for adding class members (both instance and static).

### What is added and what is not added?

What is added:
- All members, both instance and static,
- All fields, both public and static

With the following exceptions:
- Private fields are not added, otherwise that would trivially violate encapsulation.
- Only **own members** are added, not inherited ones. Authors can manually spread inherited members if they need to.

The following is TBD:
- Constructors
- Static initialization blocks
- Decorators
- Internal properties (e.g. `[[ Call ]]`). Likely handled on a case-by-case basis.

Class spread syntax is not a way to address all class partial use cases, it’s a low-level feature to make tightly coupled use cases easier to manage.
As a low-level feature, it does allow for some nonsensical patterns:

```js
class A {
	#foo = 1;
	get foo() { return this.#foo }
}
class B { ...A }

const b = new B();
console.log(b.foo);
// TypeError: Cannot read private member #foo from an object whose class did not declare it
```

### Differences from object spread syntax

The main difference from object spread syntax is that descriptors are preserved.
This means that accessors remain accessors, they are not copied like regular properties.

### What about `super`?

Ideally, `super` should resolve based on the new class hierarchy.
But given that `super` is lexically bound, that might be tricky.

### Naming collisions

Just like other uses of spread syntax, the semantics would be essentially those of a macro: any naming collisions are resolved via a last-one-wins policy, as if they have been manually specified twice.
As such, shadowed fields are fully overwritten and are not recoverable.

### References

References are preserved, but there is no way to disentangle where a member is coming from with certainty.

I.e. in the earlier example, `B.prototype.foo === A.prototype.foo` is true, but it is not possible to know whether this is a result of the spread syntax or manual assignment.

### What can be spread?

Any class can be spread, including built-in ones (though that is unlikely to be very useful).
