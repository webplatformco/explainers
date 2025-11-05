# Customizable `[[Call]]`

> [!NOTE]
> This is a work in progress and not yet ready for review.

## Motivation

Most internal properties can be tweaked on existing objects, either through dedicated methods, or known symbols:

| **Internal property** | **Settable via / controlled by** | Note |
|------------------------|----------------------------------|------|
| `[[Prototype]]` | `Object.setPrototypeOf()`, `__proto__` |
| `[[Extensible]]` | `Object.preventExtensions()` | One way only: `false` → `true` not possible |
| `[[Value]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[Writable]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[Enumerable]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[Configurable]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[Get]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[Set]]` | `Object.defineProperty()`, `Object.defineProperties()` |
| `[[PrivateFieldValues]]` | ❌ | Class field initialization (not externally settable) |
| `[[Realm]]` | ❌ | Fixed on creation |
| `[[Call]]`, `[[Construct]]` | ❌ | Function definition; `Proxy` traps override behavior |
| `[[ProxyHandler]]`, `[[ProxyTarget]]` | ❌ | Created by `new Proxy()` (immutable) |
| `[[PrimitiveValue]]` | `[Symbol.toPrimitive]()` |
| `[[ToStringTag]]` | `obj[Symbol.toStringTag]` |
| `[[HasInstance]]` | `Ctor[Symbol.hasInstance]` |
| `[[IsConcatSpreadable]]` | `obj[Symbol.isConcatSpreadable]` |
| `[[Iterator]]` | `obj[Symbol.iterator]` |
| `[[AsyncIterator]]` | `obj[Symbol.asyncIterator]` |
| `[[Match]]` | `obj[Symbol.match]` |
| `[[MatchAll]]` | `obj[Symbol.matchAll]` |
| `[[Replace]]` | `obj[Symbol.replace]` |
| `[[Search]]` | `obj[Symbol.search]` |
| `[[Split]]` | `obj[Symbol.split]` |
| `[[Species]]` | `obj[Symbol.species]` |
| `[[Unscopables]]` | `obj[Symbol.unscopables]` |
| `[[Dispose]]` | `obj[Symbol.dispose]` |
| `[[AsyncDispose]]` | `obj[Symbol.asyncDispose]` |

`[[ Call ]]` is one of the few internal properties that is not customizable.
It is however exposed as a proxy trap, which means that the only way to modify it is to create a pointless (and slow) `Proxy` around the existing function, just so one can use the `apply` trap.

### Use cases

- Adding side effects (see [mutable functions](mutable-functions.md))
- Implementing [Aspect-oriented programming (AOP)](https://en.wikipedia.org/wiki/Aspect-oriented_programming)

## Proposal

Add a new known symbol: `Symbol.call` that can be used to customize the `[[ Call ]]` internal method.
By default, `fn[Symbol.call].call(ctx, ...args)` is the same as `fn.call(ctx, ...args)`, but `fn[Symbol.call]` can be overridden while preserving references to the original function.

Built-ins for which it would be impractical to customize the `[[ Call ]]` internal method can simply define the property as non-configurable and non-writable.

For example, suppose we wanted to emulate a [mutable function](mutable-functions.md) in userland through a `Function` subclass.
If the `[[ Call ]]` internal method was customizable, we could do:

```js
class ExtensibleFunction extends Function {
	sideEffects = [];

	[Symbol.call](thisArg, ...args) {
		let result = super[Symbol.call](thisArg, ...args);
		for (const sideEffect of this.sideEffects) {
			sideEffect(result);
		}
		return result;
	}
}
```
