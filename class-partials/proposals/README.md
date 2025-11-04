# Class Partials Proposals

There are two components to addressing the requirements for class partials:
1. **Low-level primitives**: Not directly related to partials, but can be used by userland code to better implement partials.
2. **Class partials syntax**: What high-level syntax can be used to define and apply partials without the need for userland helpers?

## Low-level primitives

There are several possible primitives that can be introduced to address one or both of these components, and they are developed as separate proposals in this directory:

- [Customizable public `[[ Fields ]]`](proposals/customizable-fields.md)
- [Class spread](proposals/class-spread.md)
- [Modernize Function constructor](proposals/function-constructor.md)
- [Instance initializers / Customizable `[[Construct]]`](proposals/constructor-initializer.md)
- [Customizable `[[Call]]`](proposals/customizable-call.md)
- [Mutable Functions](proposals/mutable-functions.md)

## Class partials syntax

<!-- - [Controllers](proposals/controllers.md) -->
