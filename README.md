# Popup / Top-Layer UI Functionality

- [@mfreed7](https://github.com/mfreed7), [@scottaohara](https://github.com/scottaohara), [@BoCupp-Microsoft](https://github.com/BoCupp-Microsoft), [@domenic](https://github.com/domenic), [@gregwhitworth](https://github.com/gregwhitworth), [@chrishtr](https://github.com/chrishtr), [@dandclark](https://github.com/dandclark), [@una](https://github.com/una), [@smhigley](https://github.com/smhigley), [@aleventhal](https://github.com/aleventhal)
- March 10, 2022

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

- [Background](#background)
  - [Goals](#goals)
  - [See Also](#see-also)
- [API Shape](#api-shape)
  - [HTML Content Attribute](#html-content-attribute)
  - [Declarative Trigger (the `triggerpopup` attribute)](#declarative-trigger-the-triggerpopup-attribute)
  - [Dismiss Behavior](#dismiss-behavior)
  - [Classes of UI - Dismiss Behavior and Interactions](#classes-of-ui---dismiss-behavior-and-interactions)
  - [Events](#events)
  - [Focus Management](#focus-management)
  - [Anchoring (the `anchor` Attribute)](#anchoring-the-anchor-attribute)
  - [Display Ordering](#display-ordering)
  - [Accessibility / Semantics](#accessibility--semantics)
  - [Example Use Cases](#example-use-cases)
  - [Pros and Cons](#pros-and-cons)
- [Additional Considerations](#additional-considerations)
  - [Current Top Layer Behavior](#current-top-layer-behavior)
  - [Shadow DOM](#shadow-dom)
  - [Exceeding the Frame Bounds](#exceeding-the-frame-bounds)
  - [Eventual Single-Purpose Elements](#eventual-single-purpose-elements)
- [Other Alternatives Considered](#other-alternatives-considered)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Background

There is a need in the Web Platform for an API to create "popup UI". This is a general class of UI that always appear on top of all other content, and have both one-at-a-time and "light-dismiss" behaviors. This document proposes a set of APIs to make this type of UI easy to build.


## Goals

Here are the goals for this API (borrowed liberally from the [`<popup>` element explainer](https://open-ui.org/components/popup.research.explainer)):

* Allow any* element and its (arbitrary) descendants to be rendered on top of **all other content** in the host web application.
* Include **“light dismiss” management functionality**, to remove the element/descendants from the top-layer upon certain actions such as hitting Esc (or any [close signal](https://wicg.github.io/close-watcher/#close-signal)) or clicking outside the element bounds.
* Allow this “top layer” content to be fully styled, including properties which require compositing with other layers of the host web application (e.g. the box-shadow or backdrop-filter CSS properties).
* Allow these top layer elements to reside at semantically-relevant positions in the DOM. I.e. it should not be required to re-parent a top layer element as the last child of the `document.body` simply to escape ancestor containment and transforms.
* Allow this “top layer” content to be sized and positioned to the author's discretion.
* Include an appropriate user input and focus management experience, with flexibility to modify behaviors such as initial focus.
* **Accessible by default**, with the ability to further extend semantics/behaviors as needed for the author's specific use case.
* Avoid developer footguns, such as improper stacking of dialogs and popups, and incorrect accessibility mappings.
* Avoid the need for Javascript for the common cases.

*There may need to be some limitations in some cases.


## See Also

See the [original `<popup>` element explainer](https://open-ui.org/components/popup.research.explainer), and also the comments on [Issue 410](https://github.com/openui/open-ui/issues/410) and [Issue 417](https://github.com/openui/open-ui/issues/417). See also [this CSSWG discussion](https://github.com/w3c/csswg-drafts/issues/6965) which has mostly been about a CSS alternative for top layer.

This proposal was discussed on [Issue 455](https://github.com/openui/open-ui/issues/455), which was closed as [resolved](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067).


# API Shape

This section lays out the full details of this proposal. If you'd prefer, you can **[skip to the examples section](#example-use-cases) to see the code**.

## HTML Content Attribute

A new content attribute, **`popup`**, controls both the top layer status and the dismiss behavior. There are several allowed values for this attribute:

* **`popup=popup`** - A top layer element following “Popup” dismiss behaviors (see below).
* **`popup=hint`** - A top layer element following “Hint” dismiss behaviors (see below).
* **`popup=async`** - A top layer element following “Async” dismiss behaviors (see below).

When a popup is visible, the `open` attribute is used to indicate this status. For example:

* `<div popup=popup>` - A popup that is not yet showing. It will be set to `visibility:hidden` by the UA stylesheet, and not rendered.
* `<div popup=popup open>` - The same popup, now visible and top-layer.

It might also be possible to specify two additional values for the `popup` attribute, for modal dialogs and fullscreen elements, such that this attribute could be used to control **all** top layer element types. This would need further exploration. If this approach is **not** taken, then there might need to be restrictions placed on when the `popup` attribute can be used. For example, it should be illegal to apply `popup=popup` to an already-modal `<dialog>`.


## Declarative Trigger (the `triggerpopup` attribute)

A common design pattern is to have an activating element, such as a `<button>`, which makes a popup visible. To facilitate this pattern, another content attribute, `triggerpopup`, will also be added. This attribute's value should be the idref of another element:


```html
<button triggerpopup=mypopup>Click me</button>
<div id=mypopup popup=popup>Popup content</div>
```


When the button is activated, the UA will add the `open` attribute value to the `<div id=mypopup>` element, causing it to be rendered top-layer. In this way, no Javascript will be necessary for this use case.

When the `triggerpopup` attribute is applied to an activating element, the UA may automatically map this attribute to an appropriate `aria-*` attribute, such as `aria-haspopup`, `aria-describedby` or `aria-expanded`. There will need to be further discussion with the ARIA working group to determine the exact ARIA semantics, if any, are necessary.


## Dismiss Behavior

For elements that are displayed on the top layer via this API, there are a number of “triggers” that can cause the element to be removed from the top-layer. These fall into three main categories:



* **One at a Time.** Another element being added to the top-layer causes others to be removed. This is typically used for “one at a time” type elements: when one popup opens, other popups should be closed, so that only one is on-screen at a time. This is also used when “more important” top layer elements get added. For example, fullscreen elements should close all open popups.
* **Light Dismiss**. User action such as clicking outside the element, hitting Escape, or causing keyboard focus to leave the element. This is typically called “light dismiss”.
* **Other Reasons**. Because the top layer is a UA-managed resource, it may have other reasons (for example a user preference) to forcibly remove elements from the top layer.

In all such cases, the UA manages the removal of elements from the top layer, by forcibly removing the `open` attribute from the element. This both removes the element from the top layer and renders the element `visibility:hidden`.

The rules the UA uses to manage these interactions depends on the element types, and this is described in the following section.


## Classes of UI - Dismiss Behavior and Interactions



* Popup (**`popup=popup`**)
    * When opened, force-closes other popups and hints. An exception is ancestor popups, defined via DOM hierarchy, anchor attribute, or triggerpopup attribute.
    * It would generally be expected that a popup of this type would either receive focus, or a descendant element would receive focus when invoked.
    * Dismisses on Esc, click outside, or blur.
* Hint/Tooltip (**`popup=hint`**)
    * When opened, force-closes only other hints, but leaves all other popup types open.
    * Dismisses on Esc, click outside, when no longer hovered (after a timeout), or when the anchor element loses focus.
* Async (**`popup=async`**)
    * Does not force-close any other element type.
    * Does not light-dismiss - closes via timer or explicit close action.
* Dialog (**`<dialog>.showModal()`**)
    * When opened, force-closes popup, hint, and async.
    * Dismisses on Esc
* Fullscreen (**`<div>.requestFullscreen()`**)
    * When opened, force-closes popup, hint, async, and (with spec changes) dialog
    * Dismisses on Esc

<table>
  <tr>
   <td>
   </td>
   <td>
   </td>
   <td colspan="5" >
Second element
   </td>
  </tr>
  <tr>
   <td>
   </td>
   <td>
   </td>
   <td>Fullscreen
   </td>
   <td>Modal Dialog
   </td>
   <td>Popup
   </td>
   <td>Hint
   </td>
   <td>Async
   </td>
  </tr>
  <tr>
   <td rowspan="5" >First Element
   </td>
   <td>Fullscreen
   </td>
   <td>Close
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
  </tr>
  <tr>
   <td>Modal Dialog
   </td>
   <td>Close*
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
  </tr>
  <tr>
   <td>Popup
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
  </tr>
  <tr>
   <td>Hint
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Leave
   </td>
  </tr>
  <tr>
   <td>Async
   </td>
   <td>Close
   </td>
   <td>Close
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
   <td>Leave
   </td>
  </tr>
</table>

*Not current behavior


## Events

There is a [general desire](https://github.com/openui/open-ui/issues/342) to receive events indicating that an element has entered or left the top layer via this API. These events can be used, for example, to populate content for a popup just in time before it is shown, or update server data when it closes.

It would seem most natural and powerful to define two events:



* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.

Neither of these events would be cancellable. Both events should be fired synchronously, so that these event handlers could be used to change rendering (such as adding or removing content) without any flashes of differently-styled content.

It would also seem natural that these events would be fired for `<dialog>` elements and fullscreen elements, as they transition into and out of the top layer.


## Focus Management

Elements that “go” into the top layer sometimes need to move the focus to that element, and sometimes don't. For example, a modal `<dialog>` gets automatically focused because a dialog is something that requires immediate focus and attention. On the other hand, a `<div popup=hint>` doesn't receive focus at all (and is typically not expected to contain focusable elements). Similarly, a `<div popup=async>` should not receive focus (even if focusable) because it is meant for out-of-band communication of state, and should not interrupt a user's current action. Additionally, if the top layer element **should** receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus first. For these reasons, there need to be mechanisms available for elements to control focus in the appropriate ways. The `<popup>` element proposal provided two ways to modify focus behavior: the `delegatesfocus` and `autofocus` attributes. The `autofocus` attribute is [already a global attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autofocus), so it'd just need to be updated to clarify that the autofocus behavior happens for top layer elements right when they're added to the top layer. The other attribute provided by the `<popup>` proposal,  `delegatesfocus`, either might not be necessary, or might need to be added as a global attribute:



* Applicable to any element.
* When an element with `delegatesfocus` is focused, the first focusable descendent of that element is focused instead. This is regardless of the element's top-layer status. (This can be added using the [focus delegate spec](https://html.spec.whatwg.org/#focus-delegate).)

The use cases for these attributes need to be better enumerated.


## Anchoring (the `anchor` Attribute)

A new attribute, `anchor`, should be added for all elements, whose value is an idref of an “anchoring” element. This anchor relationship is used for two behaviors:



1. Establish the provided anchor element as an “ancestor” of this popup, for the purposes of light-dismiss behavior. In other words, when a popup is shown, typically all other popups are closed. An exception would be made for all popups in the ancestor chain formed by the anchor element.
2. The referenced anchor element could be used by the [Anchor Positioning feature](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/CSSAnchoredPositioning/explainer.md).


## Display Ordering

It is possible for an HTML document to contain multiple elements with `popup=popup`. In this case, only the last such element on the page will remain open when the page is fully loaded. This is because each successive popup element will force the previous one closed via the “one at a time” behavior.


## Accessibility / Semantics

Since the `popup` content attribute can be applied to any element, and only impacts the element's presentation (top layer vs not top layer), this should not have **any semantic or accessibility impact**. I.e., the element with the `popup` attribute will keep its existing semantics and AOM representation. In the event an author needs to extend or modify a particular element's ARIA semantics, this may be done in accordance to existing allowances of [ARIA in HTML](https://w3c.github.io/html-aria/).


## Example Use Cases

This section contains several HTML examples, showing how various UI elements might be constructed using this API.

**Note:** these examples are for demonstrative purposes of how to use the `triggerpopup` and `popup` attributes. They may not represent all necessary HTML, ARIA or JavaScript features needed to fully create such components.


### Generic Popup (Date Picker)


```html
<button id=picker-button triggerpopup=datepicker>Pick a date</button>
<my-date-picker role=dialog id=datepicker popup=popup>
  ...date picker implementation...
</my-date-picker>

<!-- No script - the triggerpopup attribute takes care of activation -->
```



###  Generic popup (`<selectmenu>` listbox example)


```html
<selectmenu>
  <template shadowroot=closed>
    <button triggerpopup=listbox>Icon</button>
    <div role=listbox id=listbox popup=popup>
      <slot></slot>
    </div>
  </template>
  <option>Option 1</option>
  <option>Option 2</option>
</selectmenu>

<!-- No script - the triggerpopup attribute takes care of activation -->
```



### Hint/Tooltip


```html
<div id=hint-trigger aria-describedby=my-hint>
  Hover me
</div>
<my-hint id=hint role=tooltip popup=hint anchor=hint-trigger>
  Hint text
</my-tooltip>

<script>
  const trigger = document.getElementById('hint-trigger');
  const hint = document.querySelector('my-hint');
  trigger.addEventListener('mouseover',() => {
    // This behavior could potentially be built into a new activation
    // content attribute, like <div trigger-on-hover=my-hint>.
    hint.addAttribute(`open','');
  });
</script>
```



### Async


```html
<div role=alert>
   <my-async-container popup=async></my-async-container>
</div>

<script>
  window.addEventListener('message',e => {
    const container = document.querySelector('my-async-container');
    container.appendChild(document.createTextNode('Msg: ' + e.data));
    container.addAttribute('open','');
  });
</script>
```



## Pros and Cons

### Pros

* Solves all of the goals of the popup effort.
* Solves the [Accessibility/Semantics problem](#alternative-dedicated-popup-element).
* Allows the UA to manage the top layer, via addition and removal of the `open` attribute.
* Works on **any element**.
* Still good DX: it should be easy for developers to understand the meaning of an attribute on an element that is in the top layer.

### Cons

* In some use cases (as [articulated here](https://github.com/openui/open-ui/issues/417#issuecomment-996890656)), the use of a content attribute might cause some DX issues. For example, in the `<selectmenu>` application, we might want to make in-page vs. popup presentation an option. To achieve that via a `popup` HTML attribute, there might need to be some mirroring of attributes from the light-dom `<selectmenu>` element to internal shadow dom container elements, which makes the shadow dom replacement feature of `<selectmenu>` a bit more complicated to both use and implement.


# Additional Considerations


## Current Top Layer Behavior

There are several “ways” that an element can currently make it to the top layer:

* The full screen API (element.requestFullScreen()).
* The `<dialog>` element, shown modally (dialog.showModal()).
* Elements using the prototype `<popup>` element implementation in Chromium.

This table documents what **currently happens** in the Chromium rendering engine, which is the only one to support any top-layer elements besides fullscreen. In particular, it documents the current `<popup>` element prototype, which is just one possibility for the general “popup” API being described here.

**<span style="text-decoration:underline;">Chromium 99.x behavior</span>**


<table>
  <tr><td></td><td colspan="4" >Second Element</td></tr>
  <tr><td rowspan="4" > First Element</td><td></td><td>Full screen</td><td>Modal dialog</td><td>`&lt;popup>` element</td></tr>
  <tr><td>Full screen</td><td>The second fullscreen element kicks the first one out of the top layer. ESC closes the second fullscreen element.</td><td>Full screen stays visible, modal is displayed above fullscreen. ESC closes fullscreen <strong>first</strong>, second ESC closes dialog (<strong>backwards</strong>).</td><td>Full screen stays visible, popup is displayed above fullscreen. ESC closes fullscreen <strong>first</strong>, second ESC closes popup (<strong>backwards</strong>).</td></tr>
  <tr><td>Modal dialog</td><td>Full screen is displayed on top of modal, both are in the top layer. ESC closes full screen, second ESC closes dialog.</td><td>Both dialogs are shown, first under second. First ESC closes second dialog, then first.</td><td>Both are shown, dialog is under popup. Hitting ESC once closes <strong>both</strong> of them.</td></tr>
  <tr><td>`&lt;popup>` element</td><td>Full screen is displayed on top of popup, both are in the top layer. ESC closes full screen, second ESC closes popup.</td><td>The dialog opening dismisses the popup. Hitting ESC closes the dialog.</td><td>The second popup dismisses the first (assuming they're not “ancestors” via DOM tree or anchor/popup attributes). Hitting ESC closes the popup.</td></tr>
</table>


Note that the current fullscreen and `<dialog>` behavior does not prevent both types of elements from being in the top layer at once. In other words, there is currently no “one-at-a-time” managment of the top layer, other than the handling of the ESC key. In some cases the keyboard interaction pattern can be a bit confusing. For example, in several cases, hitting ESC first closes the element on the **bottom**, and a second ESC closes the element on the “top”. This is on purpose, for security reasons ([specified here](https://wicg.github.io/close-watcher/#close-signal-steps)): the ESC key must always close the fullscreen element and should not be cancellable.

Generally, [per spec](https://fullscreen.spec.whatwg.org/#new-stacking-layer), when there are multiple elements in the top layer, they are rendered in the order they were added to the top layer set. Each of these elements forms a stacking context, meaning z-index cannot be used to change this painting order.



## Shadow DOM

Some thought needs to be given to what happens if an element within a shadow root uses this API to move a shadow-DOM-contained element to the top layer. One use case of such an element would be a custom element that wraps a popup type UI element, such as a `<my-tooltip>`. Should it be “ok” for a shadow element (potentially inside even a closed shadow root) to be promoted to the top layer like this? Or should it be a requirement that the light-DOM element (e.g. the `<my-tooltip>` element) itself be the one to get promoted to the top layer?

Since the rendering output from a shadow host **is already** its shadow DOM content only, it would seem totally appropriate for shadow-contained elements to be allowed to move to the top layer, since the top layer is very similar to z-index and is just a layout convenience. It also seems considerably more ergonomic to allow this, rather than requiring the top-layer API be used at the highest light-DOM element containing the desired content. There are even cases where this wouldn't make sense or be possible, such as a web component containing the entire page, `<my-app>`. In that case, it wouldn't be possible for anything contained in the app to use the top layer API.

It would seem, from the discussion above, that any element, including shadow-contained elements, **should be allowed to use this API**.

## Exceeding the Frame Bounds

Allowing a popup/top-layer element to exceed the bounds of its containing frame poses a serious security risk: such an element could spoof browser UI or containing page content. While the [original `<popup>` proposal](https://open-ui.org/components/popup.research.explainer#goals) did not discuss this issue, the [`<selectmenu>` proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ControlUICustomization/explainer.md#security) does have a specific section at least mentioning this issue. While it is possible to allow exceeding frame bounds in some cases, e.g. with the fullscreen API, great care must be taken in these cases to ensure user safety. It would seem difficult to allow an arbitrary element to exceed frame bounds using this (popup/top-layer) proposal. But perhaps there are safe ways to allow this behavior. More brainstorming is needed.

In the interim, two use counters were added to measure how often this type of behavior might be needed. They are approximations, as they merely measure the total number of times a “popup” is shown (including `<select>` popup, `<input type=color>` color picker, and `<input type=date/etc>` date/time picker), and the total number of times those popups exceed the owner frame bounds. Data can be found here:



* Total popups shown: [0.7% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3298)
* Popups appeared outside frame bounds: [0.08% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3299)

So about 11% of all popups currently exceed their owner frame bounds. That should be considered a rough upper bound, as it is possible that some of those popups **could** have fit within their frame if an attempt was made to do so, but they just happened to exceed the bounds anyway.


## Eventual Single-Purpose Elements

There might come a time, sooner or later, where a new HTML element is desired which combines strong semantics and purpose-built behaviors. For example, a `<tooltip>` or `<listbox>` element. Those elements could be relatively easily built via the APIs proposed in this document. For example, a `<tooltip>` element could be defined to have `role=tooltip` and `popup=hint`, and therefore re-use this Popup API for always-on-top rendering, one-at-a-time management, and light dismiss. In other words, these new elements could be *explained* in terms of the lower-level primitives being proposed for this API.


# Other Alternatives Considered

To achieve the [goals](#goals) of this project, a number of approaches could have been used:

* An HTML content attribute (this proposal).
* A dedicated `<popup>` element.
* A CSS property.
* A Javascript API.

Each of these options is significantly different from the others. To properly evaluate them, each option was fleshed out in some detail. Please see this document for the details of that effort:

 * [Other Alternatives Considered](other_approaches.md)

That document discusses the pros and cons for each alternative. After exploring these options, the [HTML content attribute](#html-content-attribute) approach [was determined](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067) to be the best overall.
