# Class Partials: Prior Art

Authors: Lea Verou

<details open>
<summary>Contents</summary>

1. [Userland patterns](#userland-patterns)
	1. [Subclass factories (mixins)](#subclass-factories-mixins)
	2. [Controllers](#controllers)
	3. [Prototype mutations](#prototype-mutations)
	4. [Instance mutations](#instance-mutations)
2. [Other languages](#other-languages)
	1. [Multiple inheritance languages](#multiple-inheritance-languages)
	2. [Partials](#partials)


</details>

## Userland patterns

### Subclass factories (mixins)

This is the prevailing pattern for web components:

```js
const M1 = SuperClass => class M1 extends SuperClass {}
const M2 = SuperClass => class M2 extends SuperClass {}
class A extends M2(M1(SuperClass)) {}
```

Libraries:
- [mixwith](https://www.npmjs.com/package/mixwith)

The [Mixins proposal](https://github.com/tc39/proposal-mixins) proposed a declarative syntax for mixins with automatic deduplication.

Problems:
- Pollution of the inheritance chain, no separation between identity and traits
- Requires additional code to deduplicate mixins (e.g. [`@open-wc/dedupe-mixin`](https://www.npmjs.com/package/@open-wc/dedupe-mixin))


### Controllers

Another pattern for behavior sharing is to use separate objects that hold a reference to the original instance:

```js
class Controller {
  constructor(host) {
    this.host = host;
  }

  foo () { /* elided */ }

  get bar() { /* elided */ }
  set bar(value) { /* elided */ }
}

class Foo {
  constructor() {
    this.controller = new Controller(this);
  }

  foo () {
    return this.controller.foo();
  }

  get bar() {
    return this.controller.bar;
  }

  set bar(value) {
    this.controller.bar = value;
  }
}
```

The Web Components [`ElementInternals` API](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals) is an example of this pattern in the web platform.
Lit even has a [Controller primitive](https://lit.dev/docs/composition/controllers/) to facilitate this pattern.

Pros:
- Separate state and inheritance chain makes it very easy to reason about
- Can be added and removed at any time

Problems:
- No way to add API surface to the class, so use cases that need it involve a lot of repetitive glue code
- No good way to extend existing methods, e.g. to add side effects to certain lifecycle hooks (at least not out of the box).
- No way to check whether a given class implements a certain controller or not (without a shared contract about what property the controller is stored in)
- No way to add a controller to a class from the outside, the implementing class needs to create the controller itself.

### Prototype mutations

Another pattern for behavior sharing is to mutate the prototype of the implementing class:

```js
class Mixin1 {
  foo() {
    console.log('foo from Mixin1');
  }
}

class A extends B {}

// elided for brevity
import { copyOwnDescriptors } from './utils.js';

copyOwnDescriptors({
  from: Mixin.prototype, to: Class.prototype,
  exclude: ['constructor']
});
copyOwnDescriptors({
  from: Mixin, to: Class,
  exclude: ['length', 'name', 'prototype'],
});
```

Libraries:
- [Cocktail](https://github.com/onsi/cocktail)

It is notable that Cocktail attempts to address naming conflicts via composition:

> Cocktail automatically ensures that methods defined in your mixins do not obliterate the corresponding methods in your classes. This is accomplished by wrapping all colliding methods into a new method that is then assigned to the final composite object.
> [...]
> The return value of the composite function is the last non-undefined return value from the chain of colliding functions.

Issues:
- no way for a mixin to run code at element construction time or subclass definition time (at least not without some sort of convention, like e.g. an `init()` method or similar that the implementing class needs to call).
- No way to disentangle where everything comes from (for debugging, devtools, etc).
- No way to test whether a class implements a given mixin, either via `instanceof` or some other mechanism.

### Instance mutations

Another pattern is to mutate the instance of the implementing class:

```js
import Mixin from './mixin.js';
import { applyMixin } from './utils.js';

class A {
  constructor(instance) {
    applyMixin(this, Mixin);
  }
}
```

 [traits.js](https://traitsjs.github.io/traits.js-website/) applied this pattern by composing traits into a single prototype that could then be used to make instances:

 ```js
 const TraitA = {
	methodA() { console.log("A"); }
};

const TraitB = {
	methodB() { console.log("B"); }
};

const Combined = Trait.compose(TraitA, TraitB);
const obj = Object.create(Object.prototype, Combined);

obj.methodA(); // "A"
obj.methodB(); // "B"
```

While extending instances allows for more flexibility in terms of when and how to apply the mixin,
it is considerably slower and makes it harder to reason about the API from a class reference.

## Other languages

### Multiple inheritance languages

These languages facilitate partials through multiple inheritance,
but make no distinction between identity and behavior, it's just superclasses all the way down.

#### [Python](https://docs.python.org/3/howto/mixins.html)

Python doesn’t have an explicit mixin concept,
but supports multiple inheritance:
```python
class A:
	def f(self): print("A")

class M1:
	def f(self): print("M1")

class M2:
	def f(self): print("M2")

class B(A, M1, M2):
	pass
```

Notes:
- Naming collisions are resolved via a predefined precedence order(MRO) based on the order of inclusion. The first class in the list takes precedence.
- Additional classes can be added post-hoc via `__bases__` but that is considered an anti-pattern and not all base changes are legal.

#### C++

```cpp
 class M1 {};
 class M2 {};
 class A : public M1, public M2 {};
```

#### Eiffel

Apparently one of the most elegant solutions for multiple inheritance.

```eiffel
class
	M1

feature
	f
		do
			print ("M1 f%N")
		end
end


class
	M2

feature
	f
		do
			print ("M2 f%N")
		end
end


class
	A

inherit
	M1
		rename
			f as f_from_a
		end
	M2
end
```

- Supports renaming at the point of inclusion. Any references to the renamed method in the original class continue to work.
- No generic `super`; `Precursor` must specify which superclass to use (e.g. `Precursor {M1}`)


### Partials

This includes primitives like mixins, traits, protocols, interfaces, etc.

#### [TypeScript mixins](https://www.typescriptlang.org/docs/handbook/mixins.html)

TBD

#### [PHP traits](https://www.php.net/manual/en/language.oop5.traits.php)

```php
trait M1 {}
trait M2 {}

class A
{
    use M1, M2;
    // ...
}
```

#### [Dart mixins](https://dart.dev/language/mixins)

```dart
mixin M1 {}
mixin M2 {}
class A extends B with M1, M2 {}
```

Notes:
- Interesting: [`on` clause to specify intended superclass](https://dart.dev/language/mixins#use-the-on-clause-to-declare-a-superclass).
- [`mixin class`](https://dart.dev/language/mixins#class-mixin-or-mixin-class) to define a class that can also be used as a mixin.

#### [Rust traits](https://doc.rust-lang.org/book/ch10-02-traits.html) { #rust-traits }

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
```

```rust
pub trait Summary {
    fn summarize(&self) -> String  {
      String::from("(Read more...)");
    }
}

impl Summary for NewsArticle {}
```

Notes:
- Framed largely as interfaces with "default implementations".
- Mixin to class relationship defined separately from the class itself, allowing for individual overrides of the mixin's methods.

#### [Swift protocols](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html)

```swift
protocol M1 {}
protocol M2 {}
class A: B, M1, M2 {
    // ...
}
```

#### [Scala traits](https://docs.scala-lang.org/tour/traits.html)

```scala
trait M1 {}
trait M2 {}
class A extends B with M1, M2 {}
```

Classes can extend one superclass, but can have multiple traits,
and traits can extend other traits or classes.

```scala
abstract class A:
  val message: String
class B extends A:
  val message = "I'm an instance of class B"
trait C extends A:
  def loudMessage = message.toUpperCase()
class D extends B, C

val d = D()
println(d.message)  // I'm an instance of class B
println(d.loudMessage)  // I'M AN INSTANCE OF CLASS B
```

Notes:
- Naming collisions are resolved via a predefined precedence order (class linearization) based on the order of inclusion. The last trait in the list takes precedence.
- Cannot do `class A with M1 with M2`, `with` can only come after `extends`.
- Traits affect what `super.f()` resolves to (which may come from a trait or a superclass)

#### [Haskell typeclasses](https://www.haskell.org/tutorial/classes.html)

```haskell
class M1 a where
    m1 :: a -> a
class M2 a where
    m2 :: a -> a
class A a where
    a :: a -> a

class A a where
    a :: a -> a deriving (M1, M2)
```

#### [Java 8+ interfaces](https://docs.oracle.com/javase/tutorial/java/concepts/interface.html)

In Java, interfaces can contain logic, but it is framed as a "default implementation" rather than code reuse.
As a result, superclass methods take precedence over interface methods.

```java
interface M1 {
    default void m1() {
      System.out.println("M1");
    }
}
interface M2 {
    void m2(); // abstract
}

class A implements M1, M2 {
  // ...
}
```

Notes:
- `super`:
  - `super` skips interfaces and only follows the inheritance chain.
  - For interfaces, there is `InterfaceName.super.methodName()` to call a specific default implementation from the overriding method.
- Naming conflicts:
  - If two interfaces define a default method with the same signature, the class must explicitly override that method to resolve the ambiguity, otherwise compilation fails.
  - Superclasses take precedence over interfaces.


#### [Ruby mixins](https://www.ruby-lang.org/en/documentation/ruby-from-other-languages/mixin-module/)

```ruby
module M1
    def m1
        puts "M1"
    end
end

class A
  include M1
end
```

Notes:
- Separate syntax for instance methods (`include`) and class methods (`extend`).
- Despite being specified anywhere in the class body, their placement does not affect precedence order.
- `super` resolves to the same _method_ up the chain (which may come from a module or a superclass)
- In terms of precedence order, modules sit between the implementing class and its superclass. `prepend` can change that.
