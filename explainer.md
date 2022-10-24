
# Cross-root ARIA


## Authors:

* Leo Balter <lbalter@salesforce.com>
* Manuel Rego <rego@igalia.com>


## Participate

* [Issues](https://github.com/leobalter/cross-root-aria-delegation/issues)
* [GitHub repo](https://github.com/leobalter/cross-root-aria-delegation/)


## Introduction

It is critically important that content on the web be accessible. When native elements are not the solution, ARIA is the standard which allows us to describe accessible relationships among elements. Its mechanism for this is based on IDREFS, which unfortunately cannot express relationships between different DOM trees, thus creating a problem for applying these relationships across a **shadow root**.


## Motivation

Today, because of that, authors are left with only incomplete and undesirable choices:
* Observe and move ARIA-related attributes across elements (for role, etc.).
* Use non-standard attributes for ARIA features, in order to apply them to elements in a shadow root.
* Require usage of custom elements to wrap/slot elements so that ARIA attributes can be placed directly on them. This gets very complicated as the number of slotted inputs and levels of shadow root nesting increase.
* Duplicating nodes across shadow root boundaries.
* Abandoning Shadow DOM.
* Abandoning accessibility.


## Problems

There are two different kind of problems that are very related but are not exactly the same thing. Initially they can be seen as two sides of the same coin but there are some differences when looking into them.

### 1. Import ARIA attributes into the shadow Tree

For example, we have a `label` and a custom input `x-input` that has some content and a regular `input` in the shadow tree.

We want to associate the inner `input` to the external `label`, we can set `aria-labelledby` in the `x-input`, but how can we determine which element in the shadow tree should set the relationship?

```html
<label id="label">My outer label</label>
<x-input aria-labelledby="label">
  <template shadowroot="closed">
    <span>Before</span>
    <input/>
    <span>After</span>
  </template>
</x-input>
```

We'd need a way to specify that the inner `input` is labelled by the external `label`.

One way to do that would be in JavaScript using ARIA attribute reflection setting `ariaLabelledByElements` in the inner `input` directly. But that's not possible using [Declarative Shadow DOM](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md).

In this case we're looking into a way to import attributes into the shadow tree that could be used for both string (e.g. `aria-label`) and IDREF attributes (e.g. `aria-labelledby`).

### 2. Export elements from the shadow tree so they can be referred from ARIA attributes

On a similar fashion we could have a custom label `x-label` with some content and a regular `label` in the shadow tree.

In this case we want to associate the external `input` to the inner `label`, we can set `aria-labelledby` in the `input`, but again how can we determine to which element in the shadow tree set the relationship?

```html
<x-label id="label">
  <template shadowroot="closed">
    <span>Before</span>
    <label>My inner label</label>
    <span>After</span>
  </template>
</x-label>
<input aria-labelledby="label" />
```

We'd need a way to specify that the inner `label` is somehow exposed out of the shadow tree and can be referenced by the external `input`.

Here we're looking into a way to expose some elements from the shadow tree, so they can be referenced from the outside. This is only useful for IDREF attributes (e.g. `aria-labelledby`) as this won't be needed for string ones.

### Example of combination of both problems

There could be cases where you could need both things at the same time.

Example:
```html
<fancy-input aria-controls="the-listbox" aria-activedescendant="the-listbox">
  <template shadowroot="closed">
    <input>
  </template>
</fancy-input>

<fancy-listbox id="the-listbox">
  <template shadowroot="closed">
    <div role="listbox">
      <div role="option">One</div>
      <div role="option">Two</div>
    </div>
  </template>
</fancy-listbox>
```

On one side we want to import the ARIA attributes from `fancy-input` into the inner `input`. Then we want set a relationship between that inner `input` and the inner elements of `fancy-listbox`.


## Proposals

As there are two different problems, we should look into two different proposals. But as they are so related we believe it's worth discussing them together.

### 1. Import/Delegate attributes

For this problem there's the Cross-root ARIA Delegation proposal. This proposal introduces a delegation API which would allow ARIA attributes set on a custom element to be forwarded to elements inside of its shadow root.

This will add new option to [`attachShadow()`](https://dom.spec.whatwg.org/#dom-element-attachshadow) called `delegatesAriaAttributes` similar to `delegatesFocus`. While introducing a new content attribute `delegatedariaattributes` to be used in the shadow tree elements.
This will also work on declarative Sadow DOM by adding a new `shadowrootdelegatesariaattributes` content attribute for the `template` element, similar to `shadowrootdelegatesfocus`.

Using declarative Shadow DOM like in the previous examples:
```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="closed" shadowrootdelegatesariaattributes="aria-label aria-describedby">
    <input id="input" delegatedariaattributes="aria-label aria-describedby" />
    <span delegatedariaattributes="aria-label">Another target</span>
  </template>
</x-foo>
```

And using imperative Shadow DOM:
```html
<span id="foo">Description!</span>
<template id="template1">
  <input id="input" delegatedariaattributes="aria-label aria-describedby" />
  <span delegatedariaattributes="aria-label">Another target</span>
</template>
<x-foo aria-label="Hello!" aria-describedby="foo"></x-foo>
```

```js
const template = document.getElementById('template1');

class XFoo extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open", delegatesAriaAttributes: "aria-label aria-describedby" });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}
customElements.define("x-foo", XFoo);
```

In the examples above, the ARIA attributes assigned in the host `x-foo` are delegated to inner elements inside the custom element's shadow tree. Today, custom code can reflect this application but synthetically applying the aria attributes and their effects to both the host `x-foo` and its inner elements.

#### Syntax

This proposal was originally named Cross-root ARIA Delegation, as it's somehow based in the same concept than [`delegatesFocus`](https://dom.spec.whatwg.org/#dom-shadowroot-delegatesfocus). We could discuss about the final name and think if there are better alternatives for this case. Maybe "import" or maybe other words we can think about.

Apart from that the current attribute for the `template` element is very long `shadowrootdelegatesariaattributes`, maybe it could be simplified to `delegatesariaattributes`.

#### Attributes modification timing

We'll need to determine what happens when one of the delegated attributes gets modified and when we're going to set the underneath relationships in the shadow tree elements. Probably we need to couple this with [Custom elements reactions](https://html.spec.whatwg.org/multipage/custom-elements.html#concept-custom-element-reaction) and the `attributeChangedCallback`.

#### Screen readers

Having the ARIA attributes in the shadow host and setting them also in the inner elements in the shadow tree might cause that a screen reader reads it twice.

Example:
```html
<x-input aria-label="One label">
  <template shadowroot="closed" shadowrootdelegatesariaattributes="aria-label">
    <input delegatedariaattributes="aria-label"/>
  </template>
</x-input>
```

Should we somehow "remove/ignore" the delegated attributes from the shadow host? Not removing them directly, but like ignoring them from the screen reader.

#### Import/delegate multiple attributes

If we have a custom element with two `input`s and we want to import different `aria-labelledby` elements for each of them, we cannot achieve that with this proposal.

Example:
```html
<label id="label1">Label one</label>
<label id="label2">Label two</label>
<x-input>
  <template shadowroot="closed">
    <input id="input1"/>
    <input id="input2"/>
  </template>
</x-input>
```

We could not specify that `input1` is labelled by `label1` and `input2` by `label2`. In this proposal we're setting `aria-labelledby` in the custom element `x-input` directly, so we could set `aria-labelledby="label1 label2"` there. But we cannot then set the exact relationship in the inner elements.

Not sure if there are actual use cases affected by this issue.

#### Nested shadow trees

This proposal should work if we have nested shadow trees.

Example:
```html
<label id="label">My outer label</label>
<x-input aria-labelledby="label">
  <template shadowroot="closed" shadowrootdelegatesariaattributes="aria-labelledby">
    <y-input delegatedariaattributes="aria-labelledby">
      <template shadowroot="closed" shadowrootdelegatesariaattributes="aria-labelledby">
        <input delegatedariaattributes="aria-labelledby" />
      </template>
    </y-input>
  </template>
</x-input>
```


### 2. Export elements

There could be different approaches to deal with this issue.

#### 2.1. Reflection

One approach is to do something similar to Cross-root ARIA Delegation, that's what [Cross-root ARIA Reflection](https://github.com/Westbrook/cross-root-aria-reflection/) proposal does.

Example:
```html
<span aria-controls="foo" aria-activedescendant="foo">Description</span>
<x-foo id="foo">
  <template shadowroot="open" shadowrootreflectsariaattributes="aria-controls aria-activedescendant">
    <ul reflectedariaattributes="aria-controls">
      <li>Item 1</li>
      <li reflectedariaattributes="aria-activedescendant">Item 2</li>
      <li>Item 3</li>
    </ul>
  </template>
</x-foo>
```

#### 2.2. Parts

Another approach would be reusing somehow [CSS Shadow Parts](https://drafts.csswg.org/css-shadow-parts/).

From the CSS Shadow Parts spec introduction:
> This specification defines the `::part()` pseudo-element on shadow hosts, allowing shadow hosts to selectively expose chosen elements from their shadow tree to the outside page for styling purposes.

So this specs already provide a mechanism to expose certain elements from the shadow tree to the outside world so they can be styled. We're looking for something similar so those exposed elements can be referenced by IDREF attributes from the outside.

To expose those elements CSS Shadow Parts uses the [`part`](https://drafts.csswg.org/css-shadow-parts/#part-attr) attribute, if there are nested Shadow DOM then [`exportparts`](https://drafts.csswg.org/css-shadow-parts/#exportparts-attr) is also used.

Example:
```html
<span aria-controls="foo" aria-activedescendant="foo">Description</span>
<x-foo id="foo">
  <template shadowroot="open">
    <ul part="aria-controls">
      <li>Item 1</li>
      <li part="aria-activedescendant">Item 2</li>
      <li>Item 3</li>
    </ul>
  </template>
</x-foo>
```

Maybe this doesn't make a lot of sense, as `part` is a way to expose things for styling them outside. A different approach could be use a new attribute with a different name. For example `exportfor`, and set it directly on the elements indicating for which ARIA attributes the element is exported:
```html
<span aria-controls="foo" aria-activedescendant="foo">Description</span>
<x-foo id="foo">
  <template shadowroot="open">
    <ul exportfor="aria-controls">
      <li>Item 1</li>
      <li exportfor="aria-activedescendant">Item 2</li>
      <li>Item 3</li>
    </ul>
  </template>
</x-foo>
```

On a similar fashion, but using a new attribute we could define something like a `exportids` attribute similar to `exportparts`, so we could do things like:
```html
<span aria-controls="foo-innner-list" aria-activedescendant="foo-innner-item">Description</span>
<x-foo id="foo" exportids="list: foo-innner-list, item: foo-innner-item">
  <template shadowroot="open">
    <ul id="list">
      <li>Item 1</li>
      <li id="item">Item 2</li>
      <li>Item 3</li>
    </ul>
  </template>
</x-foo>
```

One problem with this idea is that `exportids` is defined in the shadow host (like `exportparts`). When using the custom element we'd need to know the internal ids, similar to how we need to know the available parts for styling. Another issue is that each time we use the custom element we have to set this, which might be not nice.

When you start dealing with nested shadow trees things get more complicated. We have to review the proposals so they can work in those cases too.


## Issues

### Generic proposal ([issue #13](https://github.com/leobalter/cross-root-aria-delegation/issues/13))

There are more IDREF attributes apart from ARIA ones.

For example the [`for`](https://html.spec.whatwg.org/multipage/forms.html#attr-label-for) attribute in `label` element.

Example:
```html
<label id="label" for="custom-input">My outer label</label>
<x-input id="custom-input">
  <template shadowroot="closed">
    <span>Before</span>
    <input/>
    <span>After</span>
  </template>
</x-input>
```

Here we cannot set the relationship between the external `label` and the inner `input` element. A similar thing can happen in the opposite direction with a custom label referencing an input outside the shadow tree.

There are other cases that also reference to other elements:
* [`list`](https://html.spec.whatwg.org/multipage/input.html#attr-input-list) attribute in `input`.
* [Pop Up](https://open-ui.org/components/popup.research.explainer) proposal by Open UI, that adds some new attributes `popuptoggletarget`, `popupshowtarget` and `popuphidetarget`.

Should we look for a solution that is generic enough to cover these cases too?


## References

* [Public summary from WCCG](https://w3c.github.io/webcomponents-cg/#cross-root-aria)
* Related issues:
  * [WICG/aom#169](https://github.com/WICG/aom/issues/169)
  * [WICG/aom#107](https://github.com/WICG/aom/issues/107)
  * [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
  * [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

