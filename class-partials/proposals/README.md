# Class Partials Proposals

There are two components to addressing the requirements for class partials:
1. **Function side effects**: How to extend the methods in the implementing class with new behavior? Any design that is not based on inheritance will need a way to compose functions.
2. **Class partials syntax**: How to modularize class behavior and API so it can be managed across multiple components?

There are several possible primitives that can be introduced to address one or both of these components, and they are developed as separate proposals in this directory:

- [Mutable Functions](proposals/mutable-functions.md)
- [Modernize Function constructor](proposals/function-constructor.md)
- [Instance initializers / Customizable `[[Construct]]`](proposals/constructor-initializer.md)
- [Customizable `[[Call]]`](proposals/customizable-call.md)
- [Class spread](proposals/class-spread.md)
<!-- - [Controllers](proposals/controllers.md) -->
