# Class public field introspection

## Motivation

Public class fields make a common pattern of declaring instance properties more declarative.

However, while static class fields can be trivially introspected by looking at properties of the class constructor, there is no way to introspect instance fields from a class reference (without creating an instance).

## Proposal

### Core idea

A new known symbol that provides read-only or read-write access to (a subset of) a class's  internal `[[ Fields ]]` slot.

### Design decisions

#### Public only or private too?

It may be simpler to expose **public fields only**, since that’s what most introspection use cases need.

#### Instance only or static too?

Since static fields can already be introspected (though less cleanly) it may be worth only exposing **instance fields** for simplicity, as then the data structure could basically be a map of field names to initializers, with no other metadata.

OTOH, including metadata is a more extensible design, in which case there is no benefit in excluding static fields.

#### Read-only or read-write?

There is a spectrum between purely read-only access (which still addresses the use cases around introspection) and a completely mutable data structure, which allows adding fields, removing fields, or modifying initializers.

While it’s unclear whether there are use cases for a fully mutable data structure, the ability to **add** fields would make this incredibly more powerful, and address many of the use cases for [constructor side effects](constructors.md#constructor-side-effects) while sidestepping the issues around extending built-ins, since built-in classes are not created via `ClassDefinitionEvaluation` and thus have no `[[ Fields ]]` internal slot.

One question in that case is how to expose an append-only _List_.
Possibly an array-like object with a `push` method?
Or would a completely exotic iterable with an `append` method be better?
If array-like, would it include mutation methods, and just throw or fail silently?
Would an author-facing `AppendOnlyList` primitive be overkill?
Or perhaps an array with all its numerical properties defined as [[Writable]]: **false**, and [[Configurable]]: **false**?

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


