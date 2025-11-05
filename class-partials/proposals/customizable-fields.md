# Customizable `[[ Fields ]]`

## Motivation

Public class fields made the common pattern of declaring instance properties more declarative.

However, while static class fields can be trivially introspected by looking at properties of the class constructor, there is no way to introspect instance fields from a class reference (without creating an instance).

This limits many metaprogramming use cases, as there is no way to figure out the shape of the class from the outside, without creating an instance.
For example, this is needed to generate accessors to implement [delegation](delegation.md), or to emulate [class spread syntax](class-spread.md) for partials.

## Proposal

### Core idea

A new known symbol that provides read-only or limited read-write access to (a subset of) a class's internal `[[ Fields ]]` slot.

### Design decisions

#### Public only or private too?

It may be simpler to expose **public fields only**, since that’s what most introspection use cases need.

#### Instance only or static too?

Since static fields can already be introspected (though less cleanly) it may be worth only exposing **instance fields** for simplicity, as then the data structure could basically be a map of field names to initializers, with no other metadata.

OTOH, including metadata is a more extensible design, in which case there is no benefit in excluding static fields.

#### Mutable?

There is a spectrum between purely read-only access (which still addresses the use cases around introspection) and a completely mutable data structure, which allows adding fields, removing fields, or modifying initializers.

While making the data structure fully mutable seems like it adds complexity without enough use cases to warrant it, the ability to **add** can be incredibly powerful, as it addresses most of the use cases for [instance initializers](constructor-initializer.md) while sidestepping the issues around extending built-ins, since built-in classes are not created via `ClassDefinitionEvaluation` and thus have no internal `[[ Fields ]]` slot.

One question in that case is how to expose an append-only _List_.
Possibly an array-like object with a `push` method?
Or would a completely exotic iterable with an `append` method be better?
If array-like, would it include mutation methods, and just throw or fail silently?
Essentially we need the opposite of `Object.preventExtensions()`: allow extensions, but freeze existing property assignments.
Basically an array with all its numerical properties defined as [[Writable]]: **false**, and [[Configurable]]: **false**.

#### Potential known symbol name

Depending on the design decisions above, the following are potential known symbol names:

- `Symbol.fields`
- `Symbol.instanceFields`
- `Symbol.publicFields`
- `Symbol.publicInstanceFields`

### Strawman

_Just for the sake of getting the discussion started._

The initial value of `Symbol.publicFields` is the well-known symbol `%Symbol.publicFields%`.

This property has the attributes { [[Writable]]: **false**, [[Enumerable]]: **false**, [[Configurable]]: **false** }.

It is an accessor on the class constructor function object, so it’s computed from `[[ Fields ]]` and can stay consistent with decorators or other transforms that might reorder/augment during definition time.

The getter returns an append-only _List_ of _Record_ objects, each with the following properties:
- `name`: The name of the field. (`string` or `Symbol`)
- `initializer`: The initializer of the field. (`Function`)
- `isStatic`: Whether the field is static. (`boolean`)

The list is exposed using an array-like object whose numerical properties are non-writable and non-configurable.
Array methods are supported, but any attempt to modify the list will throw a `TypeError` consistent with any other attempt to modify a non-writable and non-configurable property.
