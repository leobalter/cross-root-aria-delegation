# Cross-root ARIA Delegation API explainer

source: https://github.com/WICG/webcomponents/issues/917

This proposal introduces a delegation API to allow for ARIA attributes and properties set on a Web Component to be forwarded to elements inside of its shadowroot.

This mechanism will allow users to apply standard best practices for ARIA and resolve a large margin of accessibility use cases for applications of native Web components and native Shadow DOM. This API is most suited for one-to-one delegation, but should also work for one-to-many setups. There is no mechanism for directly relating two elements in different shadowroots together, but this will still be possible manually with the element reflection API.

The proposed extension adds a new `delegates*` (e.g.: `delegatesAriaLabel`, `delegatesAriaDescribedBy`) options to the `.attachShadow` method similarly to the `delegatesFocus`, while introducing a new content attribute `auto*` (e.g.: `autoarialabel`, `autoariadescribedby`) to be used in the shadowroot inner elements. This has an advantage that it works with Declarative Shadow DOM as well, and it is consistent with `delegatesFocus`. It also works better with React-like DOM updating model where you may not necessarily have access to a DOM node right where you're creating / declaring it.

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

In the example above, the aria attributes assigned in the host `x-foo` are delegated to inner elements inside the component's shadowroot. Today, custom code can reflect this application but synthetically applying the aria attributes and their effects to both the host `x-foo` and its inner elements.

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

The attributes names are indeed long and some consideration for shorter names by removing the `shadowroot` prefix can be discussed as long the discussion is sync'ed with the stakeholders of the respective Declarative Shadow DOM proposal.

In the example above, the `<template shadowroot>` tag has the attributes `shadowrootdelegatesarialabel` and `shadowrootdelegatesariadescribedby`, matching the options in the imperative `attachShadow`:

```html
this.attachShadow({ mode: "open", delegatesAriaLabel: true, delegatesAriaDescribedBy: true });
```

This mirrors the usage of delegatesFocus as:

```html
<template shadowroot="open" shadowrootdelegatesfocus>
```

being the equivalent of:


```html
this.attachShadow({ mode: "open", delegatesFocus: true });
```

For now, consistency is being preserved, but otherwise the aria delegation attributes can be simplified as:

```html
<span id="foo">Description!</span>
<x-foo aria-label="Hello!" aria-describedby="foo">
  <template shadowroot="open" delegatesarialabel delegatesariadescribedby>
    <input id="input" autoarialabel autoariadescribedby />
    <span autoarialabel>Another target</span>
  </template>
</x-foo>
```
