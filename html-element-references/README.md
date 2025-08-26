# Improving DX of HTML Element References

Authors: Lea Verou, Noam Rosenthal

## Motivation

Many HTML attributes need to reference one or more HTML elements in the document.
This includes:

- `for`, in `<label>` and `<output>`
- `list` in `<input>`
- A host of ARIA attributes (e.g. `aria-describedby`, `aria-labelledby`, `aria-activedescendant`, `aria-controls`, `aria-details`, `aria-flowto`, `aria-owns` etc.)
- Popovers (`popovertarget`)
- Command Invokers (`commandfor`)
- Interest Invokers (`interestfor`)
- [`<template patchfor>`](https://github.com/whatwg/html/issues/11542)

It is also a frequent use case in author web components as well, for example:
- [`<wa-popover for>`](https://webawesome.com/docs/components/popover)
- [`<wa-tooltip for>`](https://webawesome.com/docs/components/tooltip)
- [`<wa-copy-button>`](https://webawesome.com/docs/components/copy-button/#copying-values-from-other-elements)

Currently, the only way to specify such references is through `id` references and using these ids to link to them in these attributes.
Some elements allow implicit associations (such as nesting a form control within a `<label>`) but most do not.

This imposes high friction, as authors then need to come up with globally unique suitable ids for elements that wouldn't otherwise have one and keep them in sync across all references.
It also introduces unnecessary error conditions; it is a common authoring mistake to change an id and forgetting to change the id references to it, or pasting a chunk of HTML and forgetting to edit all references.

This *especially* hurts accessibility, since the effects of broken references in the AT are not always obvious, and the more friction it takes to make HTML accessible, the less likely authors are to do it.

## Pain points

### Conflicts

According to an [HTTP archive query](https://docs.google.com/spreadsheets/d/19hmR_DoeQ8k8I0f_SI9QHoZy9omU_OEp5zjP8sjB954/edit?usp=sharing), roughly 45% of all HTML responses in July 2025 had multiple occurences of the same ID.
Correcting for declarative shadow DOM would only make a small dent in this number.

Since IDs are global to the document or shadow root, and resolved in document order, having the same ID multiple times in the same document is very likely to cause conflicts.
This kind of conflict can be a bug that reaches users, when e.g. labels don't target the correct input.

### Verbosity & friction

To overcome conflicts, developers have to either create some namespacing scheme that works across their entire document or shadow tree, or rely on UUIDs. At the very least, they need to add ID as a superfluous addition to an element:

```html
<form>
  <label for=searchInput>Search</label>
  <input type=search name=search id=searchInput>
</form>
```

For labels in particular there is some contextual way to do this without IDs, but this doesn't necessarily scale to all the other IDREFs.

## User research

This is a very common author pain point, and authors are pretty vocal about it.
DX-related complaints were the 3rd biggest a11y complaint in the preliminary State of HTML results 2023.

@bramus did some snowball sampling research on social media (links: [Bsky](https://bsky.app/profile/did:plc:343p6xcgmvkpz5abgezlgyep/post/3lw7kzprc6c25), [Mastodon](https://front-end.social/@bramus/115016389746283451), [X](https://x.com/bramus/status/1955283107134709886)),
asking:

> 🙋‍♂️❓
> 
> Using attributes like `for` require you to reference another element using that element’s `id`.
> 
> E.g. `<label for="x"></label> … <input id="x">`
> 
> Do you sometimes wish you had another way to refer to another element on the page, without needing to generate an id? If so, what would you want?

Most authors expressed pain points, and many wished for the ability to use CSS selectors to target elements.


@alice asked different questions:

> - What are the specific pain points around using IDs?
> - How does existing tooling address these pain points (i.e. what “cow paths” are there that we could look at?)
> - How does existing tooling fail to adequately address those pain points?
> - How are these pain points impacting authors’ abilities to author usable, accessible web pages?
> - What do authors like about IDs?
> - What design constraints should we consider for any potential solution?
> - What is the user impact of breakages in element associations? Who is impacted? How can we minimise that risk?

The results are [here](https://notes.igalia.com/XlsPwU5sQfuaWYHwhtBMtA).

TBD summarize


## Current workarounds

- React `useId()`
- Templating
- Convention-based namespacing
- Some management of IDs in a central location

### Goals
- Reduce conflicts that arise from IDs being global
- Reduce friction of linking to another element for authors, do not require tooling for reasonable DX
- Improve robustness of element references (reduce broken references)

### Non-goals

TBD

## Requirements

- Old global `id` references should be able to co-exist with the new more robust references, even on the same element.
This eases the migration path for authors and ensures existing UI libraries and other tooling that modify HTML continue to work.
Additionally, there are use cases where the referencing desired is genuinely global (e.g. a "country" field would need to autocomplete to the same list of countries everywhere and it makes sense for it to link to a single `<datalist>`).
- Ability to copy/paste fragments of HTML without breakage is a nice-to-have

## Potential solutions

Many solutions were discussed in [`whatwg/html#10143`](https://github.com/whatwg/html/issues/10143).
This section attempts to summarize the general themes.

### General querying mechanisms

These solutions are around using a general element querying mechanism to reference elements, either by querying relative to the element doing the referencing or globally, possibly with an ancestor scoping element.

#### Querying language

XPath was discussed, but consensus was largely that if we go down this path, CSS selectors are better as:
1. Authors are more familiar with them
2. They are being actively developed, and
3. There is a lot of UA code optimizing their matching

#### Syntax

- Using selectors directly in existing attributes, with a discarded prefix to specify that a selector, rather than an id, is being used, e.g. `<label for="@ & + input">`
- Using separate attributes with a consistent naming scheme, e.g. `<label forselect="& + input">` or `<label for$="& + input">`

#### Scoping

Many proposals involved using an ancestor element for scoping, to simplify the selectors necessary.
For example, to refer to the first `input` within the closest `section` one could use a general selector, but the syntax would be fairly complex: `section:not(:has(section &)):has(&) input`.

Introducing something like [lazy combinators](https://lea.verou.me/specs/lazy-combinators/) could simplify it to `& << section input`.

But being able to define `section` as the scoping element, would simplify it further to just `input`.

Scoping mechanisms could be:
- A microsyntax within the reference, e.g. `@scope (section) input`
- A separate attribute, e.g. `<label forscope="section">`
- An attribute on the scoping element, e.g. `<section refscope>`

### Identifier-based references

These solutions are around using identifiers but scoping them to an ancestor element.

#### Identifier attributes

- Continuing to use `id`, paired with a new `idscope` attribute on the scoping element, e.g. `<section idscope="section">`
- Using other existing one-to-many identifier attributes such as `name`, `class`, `itemprop` etc
- Introducing a new attribute, e.g. `ref`

#### Scoping behavior

- Fail if no element matches the reference within the scope
- Match in the closest scope first. If nothing found, continue searching in the next closest scope, all the way up to the root.

#### Scoping mechanisms

- A boolean attribute on the scoping element, e.g. `<section refscope>`
- An attribute that customizes the matching algorithm for ids, e.g. `idscope="local|global"` with no value being equivalent to `local`
- Implicit: Just improving the matching algorithm from "get the first element with this id" to something "smarter" (e.g. the same as if `idscope` had been used on every element)
- An explicit "export" attribute on the scoping element, inspired by `exportparts` e.g. `<section exportids="foo:bar, baz">`. Possibly paired with an `importids` attribute for the reverse.
- An opt-in with `exportid` attributes on the elements whose ids we want to export (with our without a value). Better locality than `exportids`.
