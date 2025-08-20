# Improving HTML Element References

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

## User research

This is a very common author pain point, and authors are pretty vocal about it.
DX-related complaints were the 3rd biggest a11y complaint in the preliminary State of HTML results 2023.

@bramus did some snowball sampling research on social media (links: [Bsky](https://bsky.app/profile/did:plc:343p6xcgmvkpz5abgezlgyep/post/3lw7kzprc6c25), [Mastodon](https://front-end.social/@bramus/115016389746283451), [X](https://x.com/bramus/status/1955283107134709886)),
asking if people would also like some alternative to using `idref`s.
While it could be argued that the question was suggestive, the pain points are very real:

> The for attribute accepting a querySelector would be handy.
>
> That way you could target `[name=foo]` or maybe even `& + *`
>
> When querySelector returns multiple fields, pick first one.
> [source](https://bsky.app/profile/rikschennink.com/post/3lwb5fvnyuc2h)

TODO: extract more quotes

[here](https://bsky.app/profile/nick-gard.bsky.social/post/3lw7mwnbl2c2n), [here](https://bsky.app/profile/emnudge.dev/post/3lw7mjpdh322e), [here](https://bsky.app/profile/awad.dev/post/3lw7un3fokk26), [here](https://bsky.app/profile/bigblind.me/post/3lw7tdyahyk2z), [here](https://bsky.app/profile/bbag.bsky.social/post/3lw7p5j2cbc2c), [here](https://front-end.social/@chriskirknielsen/115016418984599549), [here](https://front-end.social/@cwilcox808@c.im/115016999314830363), [here](https://front-end.social/@kleinfreund@mastodon.social/115016432238285991), [here](https://front-end.social/@tylersticka@social.lol/115016427837608026), [here](https://front-end.social/@mayank/115017168775422081), [here](https://x.com/DenisTRUFFAUT/status/1955316332590727651), [here](https://x.com/godwincodes/status/1955590681776591154), [here](https://x.com/myfonj/status/1955340844590067734), [here](https://x.com/vsr/status/1955293622179426741), [here](https://x.com/800147/status/1955288283631456749), …)_.




## Current workarounds

- React `useId()`
- Templating

### Goals

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
- An explicit "export" attribute on the scoping element, inspired by `exportparts` e.g. `<section exportids="foo:bar, baz>`. Possibly paired with an `importids` attribute for the reverse.
- An opt-in with `exportid` attributes on the elements whose ids we want to export (with our without a value). Better locality than `exportids`.
