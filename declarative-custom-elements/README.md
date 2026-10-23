# Yet Another Declarative Custom Elements Proposal

> [!IMPORTANT]
> This proposal is unfinished. Contributions welcome, but otherwise move along.

## Motivation

### Use cases

When discussing declarative custom elements there are two very different use cases:
1. We want declarative custom elements that make simple things easy — complex things will still require JS.
Ideally, the JS can pick up from where the HTML left off instead of rebuilding everything from scratch.
1. We want declarative custom elements as a way to make SSR of _any_ component more portable and smaller to send down the wire.

#### 1. Content-only components

Many types of components are pure content + styling, with no imperative behavior beyond what is already provided by the browser.
For example:
- Cards
- Callouts / Alerts
- Buttons
- Icons
- Badges

Addressing these has the potential to simplify these use cases quite dramatically, and reduce the pressing need for tooling even for the smallest of applications.

#### 2. Content + behavior components / SSR

An even larger set of components involves content + styling + a little behavior.
The larger the content part, the bigger the benefits from pre-rendering/SSR, since the browser can render it immediately without waiting for JS to download and execute.
For example, a component to display the page outline in a content-heavy site and highlight the current section is >90% content that can be pre-rendered, with a small amount of JS to handle the highlighting as the user scrolls.

Current solutions (DSD) depend on gzip to be effective and on tooling to generate the markup.

Often, a component may start as purely content and add behaviors later, so it's important for a solution to support these use cases as well, and provide a smooth upgrade path from the declarative rendering to the full component.

### Requirements

- The solution must work well with both light DOM and shadow DOM. Light DOM is currently avoided by WC authors due to incompatibilities with DOM diffing frameworks and the hassle of having component consumers authoring it. Both of these issues are gone if the transformation happens declaratively before anything has a chance to read the original HTML.
- Simple redirection: include attributes and element contents in arbitrary places in the rendered output as specified.
- Slots — for light DOM as well.
- It should be possible to later upgrade the element further with JS behaviors, rather than forcing a bifurcation between declarative and imperative custom elements.

### Nice to haves

- A solution that doesn't depend on custom elements at all could be utilized by frameworks as well, but would require a dramatically different design.

### Non-goals

- Computation. E.g. passing `datetime="2026-08-31"` and having the component transform it to human-readable text via `Intl.DateTimeFormat`. These types of transformations would still be in the realm of JS. We are not trying to build an expression language in HTML.
- Reconciling reactive updates to the declarative template with existing element instances that have been created from the previous template. For this to work it must be one-shot, and the API must communicate this well.



## Proposal

### Prereq: Relaxing `customElements.define()` to allow subclassing

This proposal depends on [whatwg/html#11914](https://github.com/whatwg/html/issues/11914) being accepted, otherwise there is no upgrade path to augment a declaratively defined element with imperative behaviors.
The alternative would be

### MVP

```html
<template type="element" tagname="my-card" rendermode="replace">
	<article class="card">
		<header>
			<slot name="header"></slot>
		</header>
		<div class="content">
			<slot></slot>
		</div>
		<footer>
			<slot name="footer"></slot>
		</footer>
	</article>
</template>
```

```html
<my-card>
	<h3 slot="header">This is a title</h3>
	This is a card.
</my-card>
```

renders as:

```html
<article class="card">
	<header>
		<h3>This is a title</h3>
	</header>
	<div class="content">
		This is a card.
	</div>
	<footer>
	</footer>
</article>
```

#### The `<element>` element

Most proposals try to overload the `<template>` element, but that limits what we can specify, since it conflates the template with the element definition itself,
and means all metadata needs to be specified through `<template>` attributes.
By having a separate `<element>` element, we can have any number of children for specifying metadata, which tends to work much better for one-to-many relationships.

This proposal introduces a new `<element>` element that can be used to define a custom element.
It optionally takes a `<template>` as a child.
If the child `<template>` has a `shadowrootmode` attribute, it will be used as the shadow root template,
otherwise its contents will simply be added as light DOM content.


```html
<element name="my-card">
	<template shadowrootmode="open">
		<style>
			/* Hide containers of slots that have no slotted content */
			:host > :has(slot):not(:has(slot:has-slotted)) {
				display: none;
			}
		</style>
		<article class="card">
			<header>
				<slot name="header"></slot>
			</header>
			<div class="content">
				<slot></slot>
			</div>
			<footer>
				<slot name="footer"></slot>
			</footer>
		</article>
	</template>
</element>
```

When an `<element>` is encountered by the HTML parser, a constructor class is automatically generated for it.
If a `name` attribute is specified, `customElements.define(name, ElementConstructor)` is also called to define the custom element.
The attribute can be lazily added, and `customElements.define()` will be called when the attribute is added.
Per the TAG principle that [elements do not throw when used declaratively](https://www.w3.org/2001/tag/doc/webcomponents-design-guidelines/#native-html-elements), this should never throw, even if an element is already registered with the same name.

> ISSUE: What happens when there are two definitions? Per HTML semantics, the last one wins, but they can be reordered via DOM methods.

Authors can define a subclass of `ElementConstructor` and call `customElements.define()` with the same name to layer behavior on top of the automatically generated class.

Once `customElements.define()` is called by author JS, this supersedes any declarative definition.
This is just a corollary of the definition, since any subsequent definition will not ever extend the previous class.

#### The `HTMLElementDefinitionElement` interface

`<element>` is an instance of an `HTMLElementDefinitionElement` element, which inherits from `HTMLElement`.
It includes the following instance properties:
- `name`: The name of the custom element
- `elementConstructor`: The class generated for the custom element

### Level 2

#### Forwarding attribute values: The `<attribute>` element

Many additional use cases could be handled by simply forwarding whole attribute values to one or more shadow DOM element(s).
By only dealing with whole values, we sidestep any dependency on DOM Parts and any need for interpolation syntax.

An `<attribute>` element that as a minimum takes an attribute name that the custom element will expose, and a CSS selector for the shadow DOM element(s) to redirect the attribute value to.
This can ship separately, as a Level 2 feature.

```html
<element name="image-frame">
	<attribute name="src" to="#image"></attribute>
	<template shadowrootmode="open">
		<div class="image-frame">
			<img src="" id="image">
		</div>
	</template>
</element>
```

The selector specified can match multiple elements, and the attribute value will be forwarded to all matching elements.


This association will be reactive: Changing the attribute value will update the shadow DOM element(s) content, and vice versa.
By default, no IDL property is created.
A later version may add a way to create an IDL property to reflect the attribute value.

By default the attribute is redirected with the same name.
Later versions may add a way to redirect the attribute to a different attribute, or to a text content node.



