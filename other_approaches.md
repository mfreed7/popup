# Alternative Approaches to Popup

- [@mfreed7](https://github.com/mfreed7)
- March 10, 2022

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
## Table of Contents

 - [Alternative: An HTML Content Attribute (OLD version)](#alternative-an-html-content-attribute-old-version)
 - [Alternative: Dedicated `<popup>` Element](#alternative-dedicated-popup-element)
 - [Alternative: CSS Property](#alternative-css-property)
 - [Alternative: JavaScript API](#alternative-javascript-api)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Background

This is a companion document to the main [Popup proposal](README.md), and it walks through various alternative approaches to the same problem, listing their pros and cons. Each of the approaches detailed in this document **was [rejected](https://github.com/openui/open-ui/issues/455#issuecomment-1050172067) in favor of the [HTML content attribute](README.md) approach**.


## Alternative: An HTML Content Attribute (OLD version)

This is an older version of an HTML content attribute proposal. The primary difference is that this version leaves all of the details of one-at-a-time and light dismiss behavior to the developer, to implement in Javascript. There seem to be many footguns inherent in this approach, as compared to the more [prescriptive UI classes](#classes-of-ui---dismiss-behavior-and-interactions) presented above. As such, this version was abandoned for the [current proposal](#html-content-attribute).

### API Shape

#### Top Layer Presentation

This can be implemented with a new global HTML content attribute::


```
        <my-popup toplayer>
          <content>
        </my-popup>

```



* This allows any element to be added to the top-layer.
* Admittance to the top-layer is not guaranteed or permanent - the UA can both deny an element access to the top-layer and can remove an element if needed. For example, when a modal dialog is shown or an element is made [fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Element/requestFullScreen), or a user preference desires it - those changes might “kick out” other elements residing in the top-layer.
* Some element types should likely not be allowed into the top-layer **via the API described in this document**. E.g. `<dialog>`? Or any element that is currently fullscreen?
* The toplayer attribute can be added or removed by either the developer or the UA, and in both cases, the behavior will be the same. Adding `toplayer` promotes the element to the top layer, and removing the attribute removes the element from the top layer. In the case that the UA “denies” access to the top layer, it will immediately remove the attribute.


#### The :top-layer Pseudo Class

For at least the `<popup>` use case, developers need to be able to style things differently when the `<popup>` is being shown. To enable this, we also need to add a CSS `:top-layer` pseudo-class:


```
         popup {
            opacity: 0;
            transform: translateY(50px);
            transition: all 1s;
          }
          popup:top-layer {
            opacity: 1;
            transform: translateY(0);
          }

```



* This pseudo class matches when an element is in the top-layer for any reason. That includes fullscreen or modal dialog elements.
* This pseudo does not simply match if the element has `computedStyle.position` equal to top-layer (see below), but only triggers if the element is actually in the top layer.


#### Events

There is a [general desire](https://github.com/openui/open-ui/issues/342) to receive events indicating that an element has entered or left the top layer via this API. These events can be used, for example, to populate content for a popup just in time before it is shown, or update server data when it closes. Additionally (see the section below), these events could be used to allow the application to prevent or force closing a popup in some circumstances.

It would seem most natural and powerful to define two events:

* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.

Both events would use a new TopLayerEvent interface, inheriting from [Event](https://developer.mozilla.org/en-US/docs/Web/API/Event). These events would be typically cancellable, to prevent addition or removal when desired. For some types of transition, e.g. the UA forcibly removing an element from the top layer, the events would simply **not** be cancellable.

It would also seem natural that these events would be fired for `<dialog>` elements and fullscreen elements, as they transition into and out of the top layer.


#### Dismiss Behavior

For elements that are displayed on the top layer via this API, there are a number of “triggers” that should cause the element to be removed from the top-layer. These fall into two main categories:



* **Light Dismiss**. User action such as clicking outside the element, hitting Escape, or causing keyboard focus to leave the element. This is typically called “light dismiss”.
* **One at a Time.** Another element being added to the top-layer causes others to be removed. This is typically used for “one at a time” type elements: when one popup opens, other non-ancestor popups should be closed, so that only one is onscreen at a time.

Both categories can be implemented via the events presented above, using this as the TopLayerEvent interface, which is used for all toplayer events:


```
interface TopLayerEvent : Event {
  readonly attribute DOMString reason;
};
```


To use this interface to control **light dismiss**, the `reason` attribute will contain the “reason” the element is being removed from the top layer:



* “blur” - focus was removed from the element.
* “click-outside” - a click event was received that did not fall within the element's bounding box.
* “close-signal” - a [close signal](https://wicg.github.io/close-watcher/#close-signal) was received, e.g. the ESC key. Possibly not cancellable in this case.
* “scroll” - the window was scrolled.
* “UA” - the UA is removing this element from the top layer, typically to display another element such as a fullscreen.

If the `exittoplayer` event is cancelable, then this event can be canceled when the element does not want to be dismissed via the provided `reason`. For example:


```
tooltip.addEventListener('exittoplayer',(e) => {
  if (["blur","scroll"].includes(e.reason))
    e.preventDefault(); // Blur and scroll shouldn't dismiss tooltips
});
```


To use this interface to control **one at a time** behavior, a simple querySelectorAll loop can be used within the `entertoplayer' event handler:


```
tooltip.addEventListener('entertoplayer',() => {
    for (const other of document.querySelectorAll('[toplayer]')) {
      other.removeAttribute('toplayer');
    }
});
```


In all cases above, the UA would need to fire the event handler synchronously, so that these event handlers could be used to change rendering without any flashes of differently-styled content.

If a “less Javascripty” interface was desired to expose this behavior, another way to achieve this would be to add a new CSS property, whose value is a selector:


```
.popup {
  close-other: .tooltip, .popup, my-tooltip, dialog[open];
}
```

In this case, when an element matching .popup is promoted to the top layer, anything matching the closeother property's selector value will be removed from the top layer. This would behave identically to the Javascript entertoplayer listener coded above.

In both of the above examples, any elements removed from the top layer this way will themselves receive a `exittoplayer` event, meaning they can cancel this removal.


#### Focus Management

Elements that “go” into the top layer sometimes need to move the focus to that element, and sometimes don't. For example, a modal `<dialog>` gets automatically focused because a dialog is something that requires immediate focus and attention. On the other hand, a `<tooltip>` doesn't receive focus at all (and is typically not expected to contain focusable elements). Similarly, a `<toast>` should not receive focus (even if focusable) because it is meant for out-of-band communication of state, and should not interrupt a user's current action. Additionally, if the top layer element **should** receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus first. For these reasons, there need to be mechanisms available for elements to control focus in the appropriate ways. The `<popup>` element proposal provided two ways to modify focus behavior: the `delegatesfocus` and `autofocus` attributes. The `autofocus` attribute is [already a global attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autofocus), so it'd just need to be updated to clarify that the autofocus behavior happens for top layer elements right when they're added to the top layer. The other attribute provided by the `<popup>` proposal,  `delegatesfocus`, either might not be necessary, or might need to be added as a global attribute:



* Applicable to any element.
* When an element with `delegatesfocus` is focused, the first focusable descendent of that element is focused instead. This is regardless of the element's top-layer status. (This can be added using the [focus delegate spec](https://html.spec.whatwg.org/#focus-delegate).)

The use cases for these attributes need to be better enumerated.


#### Declarative Triggering (The `popup` Attribute)

One feature provided by the `<popup>` proposal was the ability to use the `popup` content attribute on a triggering element (e.g. a button) to enable that element to show a `<popup>` when activated, without requiring Javascript.

To implement this via the HTML toplayer attribute, when the element with the popup attribute is activated by the user, the target element will simply have the `toplayer` attribute added to it by the UA, which will trigger it being placed into the top layer. In order for this to make a previously-invisible popup menu “show up”, CSS would need to be used:


```
          my-popup {
            display: none;
          }
          popup:top-layer {
            display: block;
          }
```


There could also be other such declarative attributes, e.g. `tooltip`, which trigger the popup after a hover delay, in exactly the same way as the `popup` attribute.

When the `popup` attribute is applied to an activating element, the `ariaHasPopup` property should be set appropriately by the UA.


#### Anchoring (The `anchor` Attribute)

Another feature provided by the `<popup>` proposal was the `anchor` content attribute, which provided two functionalities:



1. Establish the provided anchor element as an “ancestor” of this `<popup>` for the purposes of light-dismiss behavior. In other words, when a `<popup>` element was shown, all other `<popup>` would be closed **except** for the ancestor chain of anchor popups.
2. The referenced anchor element could be used by the [Anchor Positioning feature](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/CSSAnchoredPositioning/explainer.md).

Of these, #2 can still be achieved with a general-purpose `anchor` content attribute. It is possible that behavior #1 can also be achieved for general purpose top-layer elements. But this may be a bit odd.


#### Display Ordering and the `initiallyopen` Attribute

The `<popup>` proposal had an `initiallyopen` attribute which could be used to show the `<popup>` upon page load. Within the HTML attribute solution, this is now unnecessary. Any elements that contain the `toplayer` attribute when the document is loaded will be shown in the top layer by default.

As with the original proposal, if multiple such elements are found on a page, and they make use of the ``other-top-layer-element`` light dismiss trigger, only the last such element on the page will remain open when the page is fully loaded.


#### Accessibility / Semantics

Since the `toplayer` content attribute can be applied to any element, and only impacts the element's presentation (top layer vs not top layer), this should not have **any semantic or accessibility impact**. I.e. the element with the `toplayer` attribute will keep its existing semantics and AOM representation.


### Pros and Cons


#### Pros



* Solves the [Accessibility/Semantics problem](#alternative-dedicated-popup-element).
* Solves the [“removal from top layer” problem](#dismiss-behavior-2) inherent in the CSS solution. When the element is removed from the top layer, the UA can also remove the content attribute.
* Solves the [issue with the popup declarative attribute](#declarative-triggering-the-popup-attribute-1) in the CSS solution.
* Solves the [potential display ordering issues](#display-ordering-and-the-initiallyopen-attribute-1) raised by the CSS solution.
* Works on any element.
* Still good DX: it should be easy for developers to understand the meaning of an attribute on an element that is in the top layer.

#### Cons

* In some use cases (as [articulated here](https://github.com/openui/open-ui/issues/417#issuecomment-996890656)), the use of a content attribute might cause some DX issues. For example, in the `<selectmenu>` application, we might want to make in-page vs. popup presentation an option. To achieve that via a `toplayer` HTML attribute, there might need to be some mirroring of attributes from the light-dom `<selectmenu>` element to internal shadow dom container elements, which makes the shadow dom replacement feature of `<selectmenu>` a bit more complicated to both use and implement.


## Alternative: Dedicated `<popup>` Element

The initial [proposal](https://open-ui.org/components/popup.research.explainer) describes, in detail, the dedicated `<popup>` element approach. However, in several discussions ([1](https://github.com/w3ctag/design-reviews/issues/680#issuecomment-943472331), [2](https://github.com/openui/open-ui/issues/410), [3](https://github.com/openui/open-ui/issues/417#issuecomment-985541825), and more) the OpenUI community brought up several accessibility concerns:

* **Semantic definition of `<popup>`.** The first question [raised](https://github.com/w3ctag/design-reviews/issues/680#issuecomment-941533235) about `<popup>` was “what is the semantic definition of a `<popup>`?”. There are several possibilities, and the answer depends on the use case. One potential answer is that a `<popup>` is quite similar to (and so could re-use the semantic definition of) a non-modal dialog. But for some use cases, e.g. listboxes and action menus, a non-modal dialog is not the correct semantic, nor would the accessibility mappings associated with a non-modal dialog be expected or desired for these use cases. (See [this spreadsheet](https://docs.google.com/spreadsheets/d/1v4RXKrvp-txG477GNH42gFSzX_HkRYJk-eap6OgAorU/edit#gid=0) for a list of possible use cases and their potential semantics/roles). Therefore, to have a “general purpose” `<popup>` element, the documentation would need to clearly prohibit these “different” use cases, and instead point to other elements (many of which don't yet exist, and might never exist).
* **Default ARIA role**. Quite related to the above point, but separate and nuanced, is the question of the default ARIA role for the new `<popup>` element. One suggestion for the generic use case is “role=dialog” (non-modal), another is “role=group” (which is fairly generic), and a third is no role at all (akin to a `<div>`). There are differences of opinion ([1](https://github.com/openui/open-ui/issues/417#issuecomment-991383119),[2](https://github.com/openui/open-ui/issues/417#issuecomment-996179842),[3](https://github.com/openui/open-ui/issues/417#issuecomment-996247945)) about which of these make the most sense, and the accessibility community is quite concerned that we might do harm to the AT user community. There does not seem to be one default role that will “work” in the various “generic” use cases for the `<popup>` element approach. They do seem to strongly agree that there are some use cases (e.g. listbox) for which we **must not** use a generic `<popup>` element. Generally, picking a single role for a “generic” `<popup>` violates the goal of [“Accessible by default”](#goals).

The above accessibility/semantic issues were discussed numerous times, both in the issues linked above, and at live OpenUI meetings ([1](https://github.com/openui/open-ui/issues/410#issuecomment-948874366),[2](https://github.com/openui/open-ui/issues/410#issuecomment-948975506),[3](https://github.com/openui/open-ui/issues/410#issuecomment-961390034),[4](https://github.com/openui/open-ui/issues/410#issuecomment-973271554),[5](https://github.com/openui/open-ui/issues/417#issuecomment-984957942),[6](https://github.com/openui/open-ui/issues/417#issuecomment-996213305)). We were never able to make progress towards a resolution to go forward with a generic `<popup>` element, due to these specific concerns.

The **recommendation from the AX community is to separate the behavior from the element**. By not tying the behavior to a specific `<popup>` element with defined semantics and ARIA mappings, and instead using another mechanism to invoke the presentation/behavior, we will be able to sidestep all of the above problems.

In addition to the AX concerns discussed above, this approach (a dedicated `<popup>` element) generally violates the [separation of content and presentation](https://en.wikipedia.org/wiki/Separation_of_content_and_presentation). The `<popup>` element comes with inseparable presentational qualities such as being presented on top of all other content. Perhaps this is really the fundamental reason for the semantic and accessibility issues we're encountering?

Finally, the current `<popup>` element proposal does not support the Hint or Async use cases, at least due to their requirements to not dismiss other top layer elements when shown.


### Overall Pros and Cons


#### Pros

* Good DX: An HTML element is easy to understand and use.
* It is relatively easy to write down exactly how the `<popup>` element should interact with the other top-layer elements, because it is a single element.

#### Cons

* Unclear that there is a solution to the [Accessibility/Semantics problem](#alternative-dedicated-popup-element).
* Ties the top-layer presentation to the HTML structure. I.e. to get something to be top-layer, it must be “wrapped” in a `<popup>` element.
* Does not support more general use cases, such as Hint and Async. Those would require other new elements to be proposed.


## Alternative: CSS Property

This approach, on its face, seems very simple and elegant. However, there are two very significant downsides to this approach:
 * A “dual class” top layer will need to be created, with `<dialog>` and fullscreen always above “developer” top layer elements. That **precludes using a popup** on top of a dialog/fullscreen.
 * In this approach, light dismiss and one-at-a-time behavior cannot be built into a CSS property, and **must be implemented in JavaScript**.

For these two reasons, this approach was abandoned in favor of the [current HTML attribute proposal](#html-content-attribute).


### API Shape


#### Top Layer Presentation

This can be implemented with a new value for the CSS position property:


```
        popup {
          position: top-layer;
        }

```



* This allows any element to be added to the top-layer.
* In order to use CSS to control top layer access, there would no longer be a way to deny or remove an element from the top layer, for example when a modal dialog or fullscreen element needs to be shown. Therefore, this “developer top layer” would need to be located **underneath** the “UA top layer”, so that there was never any need to remove elements from the developer top layer.
* Some element types should likely not be allowed into the developer top-layer, such as already-modal `<dialog>`s and fullscreen elements.
* Importantly, in such a “two class” top layer world, this new CSS property could never be used to display top layer UI on top of UA top layer elements such as modal dialogs and fullscreen elements. This seems like a rather large downside.


#### The :top-layer Pseudo Class

If access to the top layer was controlled via CSS, there could not be a CSS selector to select items in the top layer. That would cause circularity problems, such as:


```
div { top-layer: yes }
div:top-layer { top-layer: no }
```


However, if control of the top layer was provided via CSS, the pseudo class would probably be superfluous. Any desired properties could just be placed into the existing selectors that trigger `top-layer:yes/no`.


#### Dismiss Behavior

As mentioned above, a CSS-based approach precludes any way for the UA to “force” elements out of the top layer. Therefore, to implement the light dismiss and one-at-a-time functionality discussed in [this section](#dismiss-behavior-1), a similar event-based approach would need to be used. In this case, there would need to be three events:



* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.
* `lightdismisstrigger`: fired on the element when there was a possible light dismiss trigger, such as a user clicking outside the element, or hitting Esc.

To implement light dismiss, the `lightdismisstrigger` event would include a `reason` field that could be used to take action:


```
interface LightDismissEvent : Event {
  readonly attribute DOMString reason;
};
```


This event and reason field would be nearly identical to the one [described here](#dismiss-behavior-1). It could be used to modify the top layer status, e.g.:


```
tooltip.addEventListener('lightdismisstrigger',(e) => {
  if (["click-outside","close-signal"].includes(e.reason))
    e.target.classList.remove('toplayer'); // Esc and click should dismiss
});
```


To implement one at a time behavior, the `entertoplayer` event can be used:


```
tooltip.addEventListener(entertoplayer',() => {
    for (const other of document.querySelectorAll('.tooltip')) {
      other.classList.remove('tooltip');
    }
});
```


Some comments:



* The `<popup>` proposal included the concept of “nested” popups - one popup could contain a nested popup (declared via DOM hierarchy, `popup` attribute, or `anchor` attribute), which would keep the parent popup from light-dismissing when the child popup opened. This might still be achievable via the same mechanisms. If we'd like this aspect of the behavior to also be configurable, perhaps the `dismiss-other` property can have an optional “except-nested” token that prevents nested elements from being dismissed. This likely needs more thought.


#### Focus Management

See [this section](#focus-management-1).


#### Declarative Triggering (The `popup` Attribute)

One feature provided by the `<popup>` proposal was the ability to use the `popup` content attribute on a triggering element (e.g. a button) to enable that element to show a `<popup>` when activated, without requiring Javascript.

To implement this in a CSS-only context, the only way I can think of would be to modify the inline `style` attribute for the target element to include `position:top-layer`. That would be somewhat odd, though it would work, and would be relatively straightforward to implement. I'm not sure there are other similar precedents in the Web Platform.

If this feature is maintained (via modifying inline styles or some other way), the `ariaHasPopup` property should be set appropriately.


#### Anchoring (The `anchor` Attribute)

The anchoring solution here would seem to be identical to the [Attribute solution for anchoring](#anchoring-the-anchor-attribute-1).


#### Display Ordering and the `initiallyopen` Attribute

Because of the light-dismiss behaviors, there needs to be a well-defined order of operations for adding elements to the top-layer. If multiple elements have `position:top-layer` and `light-dismiss:all`, then only the **last** of these elements should be eventually shown. It would seem that “last” in this context is determined by tree order, but there might be some odd scenarios. This needs more thought.

The `<popup>` proposal had an `initiallyopen` attribute which could be used to show the `<popup>` upon page load. This would seem to be unnecessary, as this could be easily achieved using author CSS with `element {position: top-layer}`, assuming the ordering issues above are resolved.


#### Accessibility / Semantics

Importantly, as the above proposals are entirely CSS-based presentational properties, none of them have **any semantic or accessibility impact**. I.e. the element being styled with `position: top-layer` will keep its existing semantics and AOM representation.


### Overall Pros and Cons


#### Pros



* Solves the [Accessibility/Semantics problem](#alternative-dedicated-popup-element).
* Good DX: CSS is easy to understand and use.
* Works on any element.


#### Cons

* This requires a “two class” top layer, with UA top layer elements such as modal `<dialog>` and fullscreen elements always on top of “developer” top layer elements. This means these two UI features are incompatible.
* Unclear how to implement the `popup` declarative activation feature.


## Alternative: JavaScript API

Generally, a JavaScript API is less preferable than an HTML/CSS based solution. And specifically, this approach suffers some of the same problems that the CSS approach has, namely that most of the one-at-a-time and light dismiss behavior is completely left to the developer to get right. Because this approach is more difficult to understand and code correctly, and doesn't offer any advantages relative to the HTML attribute solution, this approach was not used.


### API Shape

#### Top Layer Presentation

This can be implemented with a new Javascript function on Element::


```
<script>
const myPopup = document.getElementById('foo');
myPopup.requestTopLayer();
</script>
```

* This API is quite similar to the [requestFullScreen() API](https://fullscreen.spec.whatwg.org/#ref-for-dom-element-requestfullscreen%E2%91%A0).
* This allows any element to be added to the top-layer via Javascript.
* Similar to the other options presented in this document, admittance to the top-layer is not guaranteed or permanent - the UA can both deny an element access to the top-layer and can remove an element if needed. For example, when a modal dialog is shown or an element is made [fullscreen](https://developer.mozilla.org/en-US/docs/Web/API/Element/requestFullScreen) - those changes might “kick out” other elements residing in the top-layer.
* Some element types should likely not be allowed into the top-layer. E.g. `<dialog>`? Or any element that is currently fullscreen?


#### The :top-layer Pseudo Class

See [this section](#the-top-layer-pseudo-class).

#### Dismiss Behavior

It would probably make the most sense to expose the various light dismiss options via an options bag on the `requestTopLayer()` function call:


```
<script>
const myPopup = document.getElementById('foo');
const flags = new DismissFlags();
flags.scroll = true;
flags.blur = true;
flags.dismissOther = "tooltip";
myPopup.requestTopLayer({flags});
</script>
```


The [same questions apply](#dismiss-behavior-1) here about which dismissal options to support, and how to define them rigorously.


#### Focus Management

While the [same comments from the Attribute section](#focus-management-1) might apply here, it would also be possible to expose the default focus behavior as an option in the `requestTopLayer()` function call:


```
            <script>
const myPopup = document.getElementById('foo');
            myPopup.requestTopLayer({delegatesFocus: true});
</script>
```


It would seem that the content attribute based solution [presented in the Attribute section](#focus-management-1) provides a more general purpose API that would likely be better than restricting this behavior to only the top-layer API. But perhaps there's a good reason for such a restriction.


#### Events

The [Attribute solution for events](#events-1) would seem to work equally well here. Additionally, though, the `requestTopLayer()` function could return a Promise that resolves when the element is added to the top layer, or rejects if the element is not allowed to be promoted to the top layer. Whether this is actually useful would depend on whether the `requestTopLayer()` function promotes elements to the top layer synchronously or asynchronously.


#### Declarative Triggering (The `popup` Attribute)

One feature provided by the `<popup>` proposal was the ability to use the `popup` content attribute on a triggering element (e.g. a button) to enable that element to show a `<popup>` when activated, without requiring Javascript.

To implement this via the Javascript `requestTopLayer()` API, when the element with the popup attribute is activated by the user, the UA will simply call the `requestTopLayer()` function on the target element. Akin to the [HTML solution](#the-top-layer-pseudo-class), CSS will need to be used if the desire is to show a previously-invisible popup menu.

There could also be other such declarative attributes, e.g. `tooltip`, which trigger the popup after a hover delay, in exactly the same way as the `popup` attribute.

When the `popup` attribute is applied to an activating element, the `ariaHasPopup` property should be set appropriately by the UA.


#### Anchoring (The `anchor` Attribute)

The [Attribute solution for anchoring](#anchoring-the-anchor-attribute-1) would seem to apply equally well to the Javascript based API. It is perhaps possible to also/instead add another option to the `requestTopLayer()` function call to indicate ancestor popups for this purpose, but that feels rather awkward.


#### Display Ordering and the `initiallyopen` Attribute

The `<popup>` proposal had an `initiallyopen` attribute which could be used to show the `<popup>` upon page load. [That original proposal](https://open-ui.org/components/popup#the-initiallyopen-attribute) should still be usable in the context of a Javascript API, with very few changes.


#### Accessibility / Semantics

Since the Javascript `requestTopLayer()` API does not change the content at all, and only impacts the element's presentation (top layer vs not top layer) after a call to `requestTopLayer()`, this will not have **any semantic or accessibility impact**. I.e. the element before and after the `requestTopLayer()` call will maintain its existing semantics and AOM representation.


### Javascript-based APIs

There is a (strong?) bias among the group toward avoiding Javascript-only APIs, as they are more difficult to use, and the designer community is typically less comfortable and experienced with Javascript as compared to HTML/CSS. If it is possible to implement this “popup” functionality without **requiring** Javascript, that will be a win.

It is possible that the `popup` declarative attribute gets around many/most of the standard use cases, and reduces the need to manually call `requestTopLayer()` from explicit Javascript. But it still seems like an HTML attribute is more easily understandable to designers.


### Overall Pros and Cons


#### Pros



* Solves the Accessibility/Semantics problem
* Solves the [“removal from top layer” problem](#dismiss-behavior-2) inherent in the CSS solution. Since neither CSS nor HTML is affected by addition/removal from the top layer, there are no issues.
* Solves the [issue with the popup declarative attribute](#declarative-triggering-the-popup-attribute-1) in the CSS solution.
* Solves the [potential display ordering issues](#display-ordering-and-the-initiallyopen-attribute-1) raised by the CSS solution.
* Solves the [`<selectmenu>` DX issues](#cons-1) inherent in the HTML content attribute solution. The `<selectmenu>` controller can use the JS API to open and close the listbox, regardless of whether the listbox is UA-provided or developer-provided.
* Works on any element.


#### Cons

* Javascript based APIs are less desirable than something based on HTML or CSS.
* Many footguns.
