# Cross-root ARIA

- [**WIP** Spec draft](https://leobalter.github.io/cross-root-aria-delegation/)

## Public summary from WCCG

https://w3c.github.io/webcomponents-cg/#cross-root-aria

**GitHub Issue(s):**

* [WICG/aom#169](https://github.com/WICG/aom/issues/169)
* [WICG/aom#107](https://github.com/WICG/aom/issues/107)
* [WICG/webcomponents#917](https://github.com/WICG/webcomponents/issues/917)
* [WICG/webcomponents#916](https://github.com/WICG/webcomponents/issues/916)

## Description

1. Shadow root encapsulation currently prevents references between elements in different roots. Cross-root ARIA references would re-enable this platform feature within shadow roots.
2. It's not possible to "forward" ARIA attributes from a custom element host into elements in the host's shadow root. For instance, a custom input that wants to allow users to customize the ARIA role, has no way to forward the role attribute to the encapsulated native input.

## Motivation

Content on the web be accessible is critically important. Making web component content accessible currently requires many complicated and only partially-successful workarounds, such as:

* Observing and moving ARIA-related attributes across elements (for role, etc.)
* Using non-standard attributes for ARIA features, in order to apply them to elements in a shadow root.
* Requiring that custom elements users wrap/slot elements so that ARIA attributes can be placed directly on them. This gets very complicated as the number of slotted inputs and levels of shadow root nesting increase.
* Duplicating nodes across shadow root boundaries
* Abandoning Shadow DOM

## Explainers

- [Cross-root ARIA Delegation API](cross-root-aria-delegation.md)
