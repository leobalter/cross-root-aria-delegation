# Cross-root ARIA Delegation API explainer

source: https://github.com/WICG/webcomponents/issues/917

Content on the web be accessible is critically important. Maintaining accessibility within web components is challenging as the shadow root encapsulation prevents references between elements in different roots. These references are fundamental to enable assistive technologies within web applications.

Today, a web application can be composed by many web components that are connected through different relationships. ARIA is the appropriate standard to describe those relationships. Unfortunately, it's not possible to natively connect ARIA attributes from elements belonging to different DOM trees belonging to the main page and one or many web components. Because of that, some complex and not fully successful workarounds need to be applied.

* Observing and moving ARIA-related attributes across elements (for role, etc.).
* Using non-standard attributes for ARIA features, in order to apply them to elements in a shadow root.
* Requiring usage of custom elements to wrap/slot elements so that ARIA attributes can be placed directly on them. This gets very complicated as the number of slotted inputs and levels of shadow root nesting increase.
* Duplicating nodes across shadow root boundaries.
* Abandoning Shadow DOM.


This proposal introduces a delegation API to allow for ARIA attributes and properties set on a Web Component to be forwarded to elements inside of its shadowroot.

This mechanism will allow users to apply standard best practices for ARIA and resolve a large margin of accessibility use cases for applications of native Web components and native Shadow DOM. This API is most suited for one-to-one delegation, but should also work for one-to-many scenarios. There is no mechanism for directly relating two elements in different shadowroots together, but this will still be possible manually with the element reflection API.

The proposed extension adds a new `delegates*` (e.g.: `delegatesAriaLabel`, `delegatesAriaDescribedBy`) options to the `.attachShadow` method similarly to the `delegatesFocus`, while introducing a new content attribute `auto*` (e.g.: `autoarialabel`, `autoariadescribedby`) to be used in the shadowroot inner elements. This has an advantage that it works with Declarative Shadow DOM as well, and it is consistent with `delegatesFocus`. Even though, it requires another set of html attributes in the declarative shadow root template. The declarative form works better with common developer paradigm where you may not necessarily have access to a DOM node right where you're creating / declaring it.

```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  #shadow-root
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
</x-foo>
```

```html
export class XFoo extends HTMLElement {
  constructor() {
    super();
    this.attachShadow({ mode: "open", delegatesAriaLabel: true, delegatesAriaDescribedBy: true });
    this.shadowRoot.appendChild(template.content.cloneNode(true));
  }
}
customElements.define("x-foo", XFoo);
```

In the example above, the ARIA attributes assigned in the host `x-foo` are delegated to inner elements inside the component's shadowroot. Today, custom code can reflect this application but synthetically applying the aria attributes and their effects to both the host `x-foo` and its inner elements.

For instance when delegating `aria-label` you want readers to know that it applies to the input, not both the input and the host. There are current workarounds include copying the attribute over and removing it from the host, but that introduces other problems.

Live example: https://glitch.com/edit/#!/delegation-via-attr?path=x-foo.js%3A24%3A0

## Usage with Declarative Shadow DOM

This extension allows usage of attributes with [Declarative Shadow DOM](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md) and won't block SSR.

Following the same rules [to add declarative options from `attachShadow`](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md#additional-arguments-for-attachshadow) such as `delegateFocus`, we should expect ARIA delegation to be used as:

```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="open" shadowrootdelegatesarialabel shadowrootdelegatesariadescribedby>
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
  </template>
</x-foo>
```

In the example above, the `<template shadowroot>` tag has the content attributes `shadowrootdelegatesarialabel` and `shadowrootdelegatesariadescribedby`, matching the options in the imperative `attachShadow`:

```javascript
this.attachShadow({ mode: "open", delegatesAriaLabel: true, delegatesAriaDescribedBy: true });
```

This mirrors the usage of delegatesFocus as:

```html
<template shadowroot="open" shadowrootdelegatesfocus>
```

being the equivalent of:


```javascript
this.attachShadow({ mode: "open", delegatesFocus: true });
```

For now, consistency is being preserved, but otherwise the ARIA delegation attributes can be simplified as:

```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="open" delegatesarialabel delegatesariadescribedby>
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
  </template>
</x-foo>
```

* * *

# Appendix

## Thoughts: attribute names are too long

The attributes names such as `shadowrootdelegates*` are very long and some consideration for shorter names by removing the `shadowroot` prefix can be discussed as long the discussion is sync'ed with the stakeholders of the respective Declarative Shadow DOM proposal.


## Public summary from WCCG

https://w3c.github.io/webcomponents-cg/#cross-root-aria

**GitHub Issue(s):**

* [WICG/aom#169](https://github.com/WICG/aom/issues/169)
* [WICG/aom#107](https://github.com/WICG/aom/issues/107)
* [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
* [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

