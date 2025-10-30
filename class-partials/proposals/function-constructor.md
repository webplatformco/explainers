# Modernize `Function` constructor

> [!NOTE]
> This is a work in progress and not yet ready for review.

The current `Function` constructor is outdated and unsafe, operating on strings.
Even when a function object is passed as the source, it is converted to a string, which is a performance and security footgun, and leads to hard to understand errors:

```js
const fn = function () { return 1 };
const fn2 = new Function(fn);
// Uncaught SyntaxError: Function statements require a function name
```

Furthermore, the constructor expects **arguments that cannot be obtained from a function instance**, which is an API design anti-pattern and makes it impossible to properly clone functions.
The only way to clone functions is by wrapping them, which hinders debugging as it adds unnecessary noise to the stack trace, or through `eval()` which has a host of security and performance issues.

It seems that an obvious solution is to **overload** the constructor to accept a function object as the source, in which case it would be cloned.
