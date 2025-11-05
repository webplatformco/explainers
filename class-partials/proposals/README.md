# Class Partials Proposals

There are two components to addressing the requirements for class partials:
1. **Low-level primitives**: Not directly related to partials, but can be used by userland code to better implement partials and later to desugar native syntax.
2. **Class partials syntax**: What high-level syntax can be used to define and apply partials without the need for userland helpers?

It seems that a good path forwards may be to start with the low-level primitives that allow authors to more easily and robustly implement the behaviors they want, and once enough patterns have emerged, we can later design a language-level syntax.

## Low-level primitives

- [Customizable public `[[ Fields ]]`](proposals/customizable-fields.md): Read-only and possibly append-only access to the public fields of a class's internal `[[ Fields ]]` slot through a new known symbol property.
- [Class spread](proposals/class-spread.md): Spread syntax for class definitions, to facilitate behavior sharing without affecting the inheritance chain.
- [Modernize Function constructor](proposals/function-constructor.md): Modernize the Function constructor to take function objects and clone them, rather than relying on serialization.
- [Instance initializers / Customizable `[[Construct]]`](proposals/constructor-initializer.md): Allow for instance initializers that run whenever a prototype is assigned and/or after the constructor has run.
- [Customizable `[[Call]]`](proposals/customizable-call.md): Allow for the `[[Call]]` internal method to be customized through a new known symbol property.
- [Mutable Functions](proposals/mutable-functions.md): A new function type: `MutableFunction` subclass of `Function` that allows for side effects that can be added and removed after the function has been created.

## Class partials syntax

TBD

<!-- - [Controllers](proposals/controllers.md) -->
