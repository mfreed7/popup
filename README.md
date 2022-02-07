
# Popup / Top-Layer UI Functionality

- @mfreed7, @scottaohara, @BoCupp-Microsoft, @domenic, @gregwhitworth, @chrishtr
- February 7, 2022

# Background

A new `<popup>` element has been [proposed](https://open-ui.org/components/popup.research.explainer), and has received significant discussion. This document aims to review that discussion, and present various alternatives to the implementation of the functionality as proposed in the explainer.


# See Also

See the [`<popup>` explainer](https://open-ui.org/components/popup.research.explainer), and also the comments on [Issue 410](https://github.com/openui/open-ui/issues/410) and [Issue 417](https://github.com/openui/open-ui/issues/417). See also [this discussion](https://github.com/w3c/csswg-drafts/issues/6965) which has mostly been about a CSS alternative for top layer.


# Goals

Here are the goals for this functionality, mostly taken from the [goals section](https://open-ui.org/components/popup.research.explainer#goals) of the original `<popup>` element proposal, but modified to describe a more general purpose functionality:



* Allow any* element and its (arbitrary) descendants to be rendered on top of **all other content** in the host web application.
* Allow this “top layer” content to be fully styled, including properties which require compositing with other layers of the host web application (e.g. the box-shadow or backdrop-filter CSS properties).
* Allow this “top layer” content to be sized and positioned to the author’s discretion.
* Include **“light dismiss” management functionality**, to remove the element/descendants from the top-layer upon certain actions such as hitting Esc (or any [close signal](https://wicg.github.io/close-watcher/#close-signal)) or clicking outside the element bounds.
* Include an appropriate user input and focus management experience, with flexibility to modify behaviors such as initial focus.
* **Accessible by default**, with the ability to further extend semantics/behaviors as needed for the author’s specific use case.
* Avoid developer footguns, such as improper stacking of dialogs and popups, and incorrect accessibility mappings.
* Avoid the need for Javascript for the common cases.

*There may need to be some limitations in some cases.


# 


# The Options

To achieve the [goals](https://open-ui.org/components/popup.research.explainer#goals) of the “popup” project, a number of approaches could be used:



* A dedicated `<popup>` element.
* An HTML content attribute.
* A CSS property.
* A Javascript API.

After significant discussion, it seems that the HTML content attribute might be the best solution. The other options are discussed in the Other Options section of this document. Here is a quick summary of the primary reason each is not an optimal solution:



* A dedicated `<popup>` element: There is no single semantic or AX role that can be used with a “generic” popup element. It will therefore cause issues for AT users. Essentially, the presentation and semantics are not sufficiently/correct separated with this solution.
* A CSS property: There are two problems. One is that a “dual class” top layer will need to be created, with `<dialog>` and fullscreen always above “developer” top layer elements. That precludes using a popup on top of a dialog/fullscreen. The other problem is that light dismiss and one-at-a-time behavior cannot be built into a CSS property.
* A Javascript API: primarily, a JS based solution is less desirable than HTML/CSS solutions. Too many things are left to the developer, leaving many potential footguns.


# API Shape - HTML Content Attribute

A new content attribute, `**popup**`, controls both the top layer status and the dismiss behavior. There are several allowed values for this attribute:



* **`popup=popup`** - A top layer element following “Popup” dismiss behaviors (see below).
* **`popup=hint`** - A top layer element following “Hint” dismiss behaviors (see below).
* **`popup=async`** - A top layer element following “Async” dismiss behaviors (see below).

When a popup is not yet visible, the `hidden` attribute is used to indicate this status. For example:



* `<div popup=popup hidden>` - A popup that is not yet showing. It will be set to  display:none and not rendered.
* `<div popup=popup>` - The same popup, now visible and top-layer.

It might also be possible to specify two additional values for the `popup` attribute, for modal dialogs and fullscreen elements, such that this attribute could be used to control **all** top layer element types. This would need further exploration. If this approach is **not** taken, then there might need to be restrictions placed on when the `popup` attribute can be used. For example, it should be illegal to apply `popup=popup` to an already-modal `<dialog>`.


# Declarative Trigger (the `triggerpopup` attribute)

A common design pattern is to have an activating element, such as a `<button>`, which makes a popup visible. To facilitate this pattern, another content attribute, `triggerpopup`, will also be added. This attribute’s value should be the idref of another element:


```
<button triggerpopup=mypopup>Click me</button>
<div id=mypopup popup=popup hidden>Popup content</div>
```


When the button is activated, the UA will remove the `hidden` attribute value from the `mypopup` element, causing it to be rendered top-layer. In this way, no Javascript will be necessary for this use case.

When the `triggerpopup` attribute is applied to an activating element, the UA should automatically add `aria-haspopup=true` to that element.


# Dismiss Behavior

For elements that are displayed on the top layer via this API, there are a number of “triggers” that can cause the element to be removed from the top-layer. These fall into three main categories:



* **One at a Time.** Another element being added to the top-layer causes others to be removed. This is typically used for “one at a time” type elements: when one popup opens, other popups should be closed, so that only one is on-screen at a time. This is also used when “more important” top layer elements get added. For example, fullscreen elements should close other popups.
* **Light Dismiss**. User action such as clicking outside the element, hitting Escape, or causing keyboard focus to leave the element. This is typically called “light dismiss”.
* **Other Reasons**. Because the top layer is a UA-managed resource, it may have other reasons (for example a user preference) to forcibly remove elements from the top layer.

In all such cases, the UA manages the removal of elements from the top layer, by forcibly removing the `hidden` attribute from the element. This both removes the element from the top layer and renders the element display:none.

The rules the UA uses to manage these interactions depends on the element types, and this is described in the following section.


# Classes of UI - Dismiss Behavior and Interactions



* Popup (`popup=popup`)
    * When opened, force-closes other popups and hints. An exception is ancestor popups, defined via DOM hierarchy, anchor attribute, or triggerpopup attribute.
    * Dismisses on Esc, click outside, or blur.
* Hint/Tooltip (`popup=hint`)
    * When opened, force-closes only other hints, but leaves all other popup types open.
    * Dismisses on Esc, click outside, when no longer hovered (after a timeout), or when the anchor element loses focus.
* Async (`popup=async`)
    * Does not force-close any other element type.
    * Does not light-dismiss - closes via timer or explicit close action.
* Dialog (`<dialog>.showModal()`)
    * When opened, force-closes popup, hint, and async.
    * Dismisses on Esc
* Fullscreen (`<div>.requestFullscreen()`)
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


* Not current behavior


# Events

There is a [general desire](https://github.com/openui/open-ui/issues/342) to receive events indicating that an element has entered or left the top layer via this API. These events can be used, for example, to populate content for a popup just in time before it is shown, or update server data when it closes.

It would seem most natural and powerful to define two events:



* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.

Neither of these events would be cancellable. Both events should be fired synchronously, so that these event handlers could be used to change rendering (such as adding or removing content) without any flashes of differently-styled content.

It would also seem natural that these events would be fired for `<dialog>` elements and fullscreen elements, as they transition into and out of the top layer.


# Focus Management

Elements that “go” into the top layer sometimes need to move the focus to that element, and sometimes don’t. For example, a modal `<dialog>` gets automatically focused because a dialog is something that requires immediate focus and attention. On the other hand, a `<div popup=hint>` doesn’t receive focus at all (and is typically not expected to contain focusable elements). Similarly, an `<div popup=async>` should not receive focus (even if focusable) because it is meant for out-of-band communication of state, and should not interrupt a user’s current action. Additionally, if the top layer element **should** receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus first. For these reasons, there need to be mechanisms available for elements to control focus in the appropriate ways. The `<popup>` element proposal provided two ways to modify focus behavior: the `delegatesfocus` and `autofocus` attributes. The `autofocus` attribute is [already a global attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autofocus), so it’d just need to be updated to clarify that the autofocus behavior happens for top layer elements right when they’re added to the top layer. The other attribute provided by the `<popup>` proposal,  `delegatesfocus`, either might not be necessary, or might need to be added as a global attribute:



* Applicable to any element.
* When an element with `delegatesfocus` is focused, the first focusable descendent of that element is focused instead. This is regardless of the element’s top-layer status. (This can be added using the [focus delegate spec](https://html.spec.whatwg.org/#focus-delegate).)

The use cases for these attributes need to be better enumerated.


# Anchoring (The `anchor` Attribute)

A new attribute, `anchor`, should be added for all elements, whose value is an idref of an “anchoring” element. This anchor relationship is used for two behaviors:



1. Establish the provided anchor element as an “ancestor” of this popup, for the purposes of light-dismiss behavior. In other words, when a popup is shown, typically all other popups are closed. An exception would be made for all popups in the ancestor chain formed by the anchor element.
2. The referenced anchor element could be used by the [Anchor Positioning feature](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/CSSAnchoredPositioning/explainer.md).


# Display Ordering

It is possible for an HTML document to contain multiple elements with `popup=popup`. In this case, only the last such element on the page will remain open when the page is fully loaded. This is because each successive popup element will force the previous one closed via the “one at a time” behavior.


# Accessibility / Semantics

Since the `toplayer` content attribute can be applied to any element, and only impacts the element’s presentation (top layer vs not top layer), this should not have **any semantic or accessibility impact**. I.e. the element with the `toplayer` attribute will keep its existing semantics and AOM representation.


# Example Use Cases

This section contains several HTML examples, showing how various UI elements might be constructed using this API.


## Generic Popup (Date Picker)


```
<button id=picker-button triggerpopup=datepicker>Pick a date</button> 
<my-date-picker role=dialog id=datepicker popup=popup hidden>
  ...date picker implementation...
</my-date-picker>
<!-- No script - the triggerpopup attribute takes care of activation -->
```



##  Generic popup (`<selectmenu>` listbox example)


```
<selectmenu>
  <template shadowroot=closed>
    <button triggerpopup=listbox>Icon</button>
    <div role=listbox id=listbox popup=popup hidden>
      <slot></slot>
    </div>
  </template>
  <option>Option 1</option>
  <option>Option 2</option>
</selectmenu>
<!-- No script - the triggerpopup attribute takes care of activation -->  
```



## Hint/Tooltip


```
<div id=hint-trigger aria-describedby=my-hint>
  Hover me
</div> 
<my-hint id=hint role=tooltip popup=hint hidden anchor=hint-trigger>
  Hint text
</my-tooltip> 
<script>
  const trigger = document.getElementById('hint-trigger');
  const hint = document.querySelector('my-hint');
  trigger.addEventListener('mouseover',() => {
    // This behavior could potentially be built into a new activation
    // content attribute, like <div trigger-on-hover=my-hint>.
    hint.removeAttribute(‘hidden’);
  });
</script>
```



## Async


```
<div role=alert>
   <my-async-container popup=async hidden></my-async-container>
</div>
<script>
  window.addEventListener('message',e => {
    const container = document.querySelector('my-async-container');
    container.appendChild(document.createTextNode('Msg: ' + e.data));
    container.removeAttribute(‘hidden’);
  });
</script>
```



# Pros and Cons


## Pros



* Solves all of the goals of the popup effort.
* Solves the [Accessibility/Semantics problem](#bookmark=id.lkz5h0jgcbkl).
* Allows the UA to manage the top layer, via changing the `popup` attribute value.
* Works on **any element**.
* Still good DX: it should be easy for developers to understand the meaning of an attribute on an element that is in the top layer.


## Cons



* In some use cases (as [articulated here](https://github.com/openui/open-ui/issues/417#issuecomment-996890656)), the use of a content attribute might cause some DX issues. For example, in the `<selectmenu>` application, we might want to make in-page vs. popup presentation an option. To achieve that via a `toplayer` HTML attribute, there might need to be some mirroring of attributes from the light-dom `<selectmenu>` element to internal shadow dom container elements, which makes the shadow dom replacement feature of `<selectmenu>` a bit more complicated to both use and implement.


# 


# Other Options We Explored


## Option: An HTML Content Attribute (OLD version)

This is an older version of an HTML content attribute proposal. It uses a lot of Javascript, rather than more prescriptive UI classes, to implement light dismiss and one-at-a-time. As such, this version was abandoned for the current proposal.

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

It would seem most natural and powerful to define four events:



* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.

Both events would use a new TopLayerEvent interface, inheriting from [Event](https://developer.mozilla.org/en-US/docs/Web/API/Event). These events would be typically cancellable, to prevent addition or removal when desired. For some types of transition, e.g. the UA forcibly removing an element from the top layer, the events would simply **not** be cancellable.


### It would also seem natural that these events would be fired for `<dialog>` elements and fullscreen elements, as they transition into and out of the top layer.


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
* “click-outside” - a click event was received that did not fall within the element’s bounding box.
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


To use this interface to control **one at a time** behavior, a simple querySelectorAll loop can be used within the `entertoplayer’ event handler:


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


In this case, when an element matching .popup is promoted to the top layer, anything matching the closeother property’s selector value will be removed from the top layer. This would behave identically to the Javascript entertoplayer listener coded above.


### In both of the above examples, any elements removed from the top layer this way will themselves receive a `exittoplayer` event, meaning they can cancel this removal.


#### Focus Management

Elements that “go” into the top layer sometimes need to move the focus to that element, and sometimes don’t. For example, a modal `<dialog>` gets automatically focused because a dialog is something that requires immediate focus and attention. On the other hand, a `<tooltip>` doesn’t receive focus at all (and is typically not expected to contain focusable elements). Similarly, a `<toast>` should not receive focus (even if focusable) because it is meant for out-of-band communication of state, and should not interrupt a user’s current action. Additionally, if the top layer element **should **receive immediate focus, there is a question about **which** part of the element gets that initial focus. For example, the element itself could receive focus, or one of its focusable descendants could receive focus first. For these reasons, there need to be mechanisms available for elements to control focus in the appropriate ways. The `<popup>` element proposal provided two ways to modify focus behavior: the `delegatesfocus` and `autofocus` attributes. The `autofocus` attribute is [already a global attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes/autofocus), so it’d just need to be updated to clarify that the autofocus behavior happens for top layer elements right when they’re added to the top layer. The other attribute provided by the `<popup>` proposal,  `delegatesfocus`, either might not be necessary, or might need to be added as a global attribute:



* Applicable to any element.
* When an element with `delegatesfocus` is focused, the first focusable descendent of that element is focused instead. This is regardless of the element’s top-layer status. (This can be added using the [focus delegate spec](https://html.spec.whatwg.org/#focus-delegate).)

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


There could also be other such declarative attributes, e.g. `tooltip`, which trigger the popup after a hover delay, in exactly the same way as the ‘popup’ attribute. 

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

Since the `toplayer` content attribute can be applied to any element, and only impacts the element’s presentation (top layer vs not top layer), this should not have **any semantic or accessibility impact**. I.e. the element with the `toplayer` attribute will keep its existing semantics and AOM representation.


### Pros and Cons


#### Pros



* Solves the [Accessibility/Semantics problem](#bookmark=id.lkz5h0jgcbkl).
* Solves the [“removal from top layer” problem](#bookmark=id.nzep20lyr6um) inherent in the CSS solution. When the element is removed from the top layer, the UA can also remove the content attribute.
* Solves the [issue with the popup declarative attribute](#bookmark=id.pb6tscmhlpdt) in the CSS solution.
* Solves the [potential display ordering issues](#bookmark=id.3g4ul4ry1hq) raised by the CSS solution.
* Works on any element.
* Still good DX: it should be easy for developers to understand the meaning of an attribute on an element that is in the top layer.


#### Cons



* In some use cases (as [articulated here](https://github.com/openui/open-ui/issues/417#issuecomment-996890656)), the use of a content attribute might cause some DX issues. For example, in the `<selectmenu>` application, we might want to make in-page vs. popup presentation an option. To achieve that via a `toplayer` HTML attribute, there might need to be some mirroring of attributes from the light-dom `<selectmenu>` element to internal shadow dom container elements, which makes the shadow dom replacement feature of `<selectmenu>` a bit more complicated to both use and implement.


## Option: Dedicated `<popup>` Element

The initial [proposal](https://open-ui.org/components/popup.research.explainer) describes, in detail, the dedicated `<popup>` element approach. However, in several discussions ([1](https://github.com/w3ctag/design-reviews/issues/680#issuecomment-943472331), [2](https://github.com/openui/open-ui/issues/410), [3](https://github.com/openui/open-ui/issues/417#issuecomment-985541825), and more) the OpenUI community brought up several accessibility concerns:



* **Semantic definition of `<popup>`.** The first question [raised](https://github.com/w3ctag/design-reviews/issues/680#issuecomment-941533235) about `<popup>` was “what is the semantic definition of a `<popup>`?”. There are several possibilities, and the answer depends on the use case. One potential answer is that a `<popup>` is quite similar to (and so could re-use the semantic definition of) a non-modal dialog. But for some use cases, e.g. listboxes and action menus, a non-modal dialog is not the correct semantic, nor would the accessibility mappings associated with a non-modal dialog be expected or desired for these use cases. (See [this spreadsheet](https://docs.google.com/spreadsheets/d/1v4RXKrvp-txG477GNH42gFSzX_HkRYJk-eap6OgAorU/edit#gid=0) for a list of possible use cases and their potential semantics/roles). Therefore, to have a “general purpose” `<popup>` element, the documentation would need to clearly prohibit these “different” use cases, and instead point to other elements (many of which don’t yet exist, and might never exist).
* **Default ARIA role**. Quite related to the above point, but separate and nuanced, is the question of the default ARIA role for the new `<popup>` element. One suggestion for the generic use case is “role=dialog” (non-modal), another is “role=group” (which is fairly generic), and a third is no role at all (akin to a `<div>`). There are differences of opinion ([1](https://github.com/openui/open-ui/issues/417#issuecomment-991383119),[2](https://github.com/openui/open-ui/issues/417#issuecomment-996179842),[3](https://github.com/openui/open-ui/issues/417#issuecomment-996247945)) about which of these make the most sense, and the accessibility community is quite concerned that we might do harm to the AT user community. There does not seem to be one default role that will “work” in the various “generic” use cases for the `<popup>` element approach. They do seem to strongly agree that there are some use cases (e.g. listbox) for which we **must not** use a generic `<popup>` element. Generally, picking a single role for a “generic” `<popup>` violates the goal of [“Accessible by default”](#bookmark=id.nhqtkq2wpf2e).

The above accessibility/semantic issues were discussed numerous times, both in the issues linked above, and at live OpenUI meetings ([1](https://github.com/openui/open-ui/issues/410#issuecomment-948874366),[2](https://github.com/openui/open-ui/issues/410#issuecomment-948975506),[3](https://github.com/openui/open-ui/issues/410#issuecomment-961390034),[4](https://github.com/openui/open-ui/issues/410#issuecomment-973271554),[5](https://github.com/openui/open-ui/issues/417#issuecomment-984957942),[6](https://github.com/openui/open-ui/issues/417#issuecomment-996213305)). We were never able to make progress towards a resolution to go forward with a generic `<popup>` element, due to these specific concerns.

The recommendation from the AX community is to separate the behavior from the element. By not tying the behavior to a specific `<popup>` element with defined semantics and ARIA mappings, and instead using another mechanism to invoke the presentation/behavior, we will be able to sidestep all of the above problems.

In addition to the AX concerns discussed above, this approach (a dedicated `<popup>` element) generally violates the [separation of content and presentation](https://en.wikipedia.org/wiki/Separation_of_content_and_presentation). The `<popup>` element comes with inseparable presentational qualities such as being presented on top of all other content. Perhaps this is really the fundamental reason for the semantic and accessibility issues we’re encountering?

Finally, as observed in the [discussion of other top layer element types](#bookmark=id.78b8u2jp4fxu), the current `<popup>` proposal does not support the Tooltip or Async use cases, at least due to their requirements to not dismiss other top layer elements when shown.


### Overall Pros and Cons


#### Pros



* Good DX: An HTML element is easy to understand and use.
* It is relatively easy to write down exactly how the `<popup>` element should interact with the other top-layer elements, because it is a single element.


#### Cons



* Unclear that there is a solution to the [Accessibility/Semantics problem](#bookmark=id.lkz5h0jgcbkl).
* Ties the top-layer presentation to the HTML structure. I.e. to get something to be top-layer, it must be “wrapped” in a `<popup>` element.
* Does not support more general use cases, such as Tooltip and Async. Those would require other new elements to be proposed.
* 


## Option: CSS Property


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

As mentioned above, a CSS-based approach precludes any way for the UA to “force” elements out of the top layer. Therefore, to implement the light dismiss and one-at-a-time functionality discussed in [this section](#bookmark=kix.v6oku3r2fwb4), a similar event-based approach would need to be used. In this case, there would need to be three events:



* `entertoplayer`: fired on the element just after it is promoted to the top layer.
* `exittoplayer`: fired on the element just after it is removed from the top layer.
* `lightdismisstrigger`: fired on the element when there was a possible light dismiss trigger, such as a user clicking outside the element, or hitting Esc.

To implement light dismiss, the `lightdismisstrigger` event would include a `reason` field that could be used to take action:


```
interface LightDismissEvent : Event {
  readonly attribute DOMString reason;
};
```


This event and reason field would be nearly identical to the one [described here](#bookmark=id.3u61lmupqocs). It could be used to modify the top layer status, e.g.:


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



* The `<popup>` proposal included the concept of “nested” popups - one popup could contain a nested popup (declared via DOM hierarchy, `popup` attribute, or `anchor` attribute), which would keep the parent popup from light-dismissing when the child popup opened. This might still be achievable via the same mechanisms. If we’d like this aspect of the behavior to also be configurable, perhaps the `dismiss-other` property can have an optional “except-nested” token that prevents nested elements from being dismissed. This likely needs more thought.


#### Focus Management

See [this section](#bookmark=id.zfcmczclil5g).


#### Declarative Triggering (The `popup` Attribute)

One feature provided by the `<popup>` proposal was the ability to use the `popup` content attribute on a triggering element (e.g. a button) to enable that element to show a `<popup>` when activated, without requiring Javascript.

To implement this in a CSS-only context, the only way I can think of would be to modify the inline `style` attribute for the target element to include `position:top-layer`. That would be somewhat odd, though it would work, and would be relatively straightforward to implement. I’m not sure there are other similar precedents in the Web Platform.

If this feature is maintained (via modifying inline styles or some other way), the `ariaHasPopup` property should be set appropriately.


#### Anchoring (The `anchor` Attribute)

The anchoring solution here would seem to be identical to the [Attribute solution for anchoring](#bookmark=id.grw24p3o02qv).


#### Display Ordering and the `initiallyopen` Attribute

Because of the light-dismiss behaviors, there needs to be a well-defined order of operations for adding elements to the top-layer. If multiple elements have `position:top-layer` and `light-dismiss:all`, then only the **last** of these elements should be eventually shown. It would seem that “last” in this context is determined by tree order, but there might be some odd scenarios. This needs more thought.

The `<popup>` proposal had an `initiallyopen` attribute which could be used to show the `<popup>` upon page load. This would seem to be unnecessary, as this could be easily achieved using author CSS with `element {position: top-layer}`, assuming the ordering issues above are resolved.


#### Accessibility / Semantics

Importantly, as the above proposals are entirely CSS-based presentational properties, none of them have **any semantic or accessibility impact**. I.e. the element being styled with `position: top-layer` will keep its existing semantics and AOM representation.


### Overall Pros and Cons


#### Pros



* Solves the [Accessibility/Semantics problem](#bookmark=id.lkz5h0jgcbkl).
* Good DX: CSS is easy to understand and use.
* Works on any element.


#### Cons



* This requires a “two class” top layer, with UA top layer elements such as modal `<dialog>` and fullscreen elements always on top of “developer” top layer elements. This means these two UI features are incompatible.
* Unclear how to implement the `popup` declarative activation feature.


## Option: Javascript API


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

See [this section](#bookmark=id.sn8ddixtl649).


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


The [same questions apply](#bookmark=id.x3h7twvyv04o) here about which dismissal options to support, and how to define them rigorously.


#### Focus Management

While the [same comments from the Attribute section](#bookmark=id.zfcmczclil5g) might apply here, it would also be possible to expose the default focus behavior as an option in the `requestTopLayer()` function call:


```
            <script>
const myPopup = document.getElementById('foo');
            myPopup.requestTopLayer({delegatesFocus: true});
</script>
```


It would seem that the content attribute based solution [presented in the Attribute section](#bookmark=id.zfcmczclil5g) provides a more general purpose API that would likely be better than restricting this behavior to only the top-layer API. But perhaps there’s a good reason for such a restriction. 


#### Events

The [Attribute solution for events](#bookmark=id.vu5017g2x0gd) would seem to work equally well here. Additionally, though, the `requestTopLayer()` function could return a Promise that resolves when the element is added to the top layer, or rejects if the element is not allowed to be promoted to the top layer. Whether this is actually useful would depend on whether the `requestTopLayer()` function promotes elements to the top layer synchronously or asynchronously.


#### Declarative Triggering (The `popup` Attribute)

One feature provided by the `<popup>` proposal was the ability to use the `popup` content attribute on a triggering element (e.g. a button) to enable that element to show a `<popup>` when activated, without requiring Javascript.

To implement this via the Javascript `requestTopLayer()` API, when the element with the popup attribute is activated by the user, the UA will simply call the `requestTopLayer()` function on the target element. Akin to the [HTML solution](#bookmark=id.j0y8xof0qwhc), CSS will need to be used if the desire is to show a previously-invisible popup menu.

There could also be other such declarative attributes, e.g. `tooltip`, which trigger the popup after a hover delay, in exactly the same way as the ‘popup’ attribute. 

When the `popup` attribute is applied to an activating element, the `ariaHasPopup` property should be set appropriately by the UA.


#### Anchoring (The `anchor` Attribute)

The [Attribute solution for anchoring](#bookmark=id.grw24p3o02qv) would seem to apply equally well to the Javascript based API. It is perhaps possible to also/instead add another option to the `requestTopLayer()` function call to indicate ancestor popups for this purpose, but that feels rather awkward.


#### Display Ordering and the `initiallyopen` Attribute

The `<popup>` proposal had an `initiallyopen` attribute which could be used to show the `<popup>` upon page load. [That original proposal](https://open-ui.org/components/popup#the-initiallyopen-attribute) should still be usable in the context of a Javascript API, with very few changes.


#### Accessibility / Semantics

Since the Javascript `requestTopLayer()` API does not change the content at all, and only impacts the element’s presentation (top layer vs not top layer) after a call to `requestTopLayer()`, this will not have **any semantic or accessibility impact**. I.e. the element before and after the `requestTopLayer()` call will maintain its existing semantics and AOM representation.


### Javascript-based APIs

There is a (strong?) bias among the group toward avoiding Javascript-only APIs, as they are more difficult to use, and the designer community is typically less comfortable and experienced with Javascript as compared to HTML/CSS. If it is possible to implement this “popup” functionality without **requiring** Javascript, that will be a win.

It is possible that the `popup` declarative attribute gets around many/most of the standard use cases, and reduces the need to manually call `requestTopLayer()` from explicit Javascript. But it still seems like an HTML attribute is more easily understandable to designers.


### Overall Pros and Cons


#### Pros



* Solves the Accessibility/Semantics problem
* Solves the [“removal from top layer” problem](#bookmark=id.nzep20lyr6um) inherent in the CSS solution. Since neither CSS nor HTML is affected by addition/removal from the top layer, there are no issues.
* Solves the [issue with the popup declarative attribute](#bookmark=id.pb6tscmhlpdt) in the CSS solution.
* Solves the [potential display ordering issues](#bookmark=id.3g4ul4ry1hq) raised by the CSS solution.
* Solves the [`<selectmenu>` DX issues](#bookmark=id.l0bk4qj8w74v) inherent in the HTML content attribute solution. The `<selectmenu>` controller can use the JS API to open and close the listbox, regardless of whether the listbox is UA-provided or developer-provided.
* Works on any element.


#### Cons



* Javascript based APIs are less desirable than something based on HTML or CSS.


# Interactions Between Top Layer Elements

There are several “ways” that an element can make it to the top layer:



* The full screen API (element.requestFullScreen()).
* The `<dialog>` element, shown modally (dialog.showModal()).
* Elements using the API proposed in this document (“popups”).

Since the top layer is managed by the UA, there should be rules for precedence and behavior when more than one such element occupies the top layer or requests access to the top layer. For example, if a modal dialog is showing, and an element requests full screen access, what should happen to the modal dialog? Leave it open (underneath the fullscreen element)? Or close it before promoting the element to fullscreen?


## Current Behavior

This table documents what **currently happens** in the Chromium rendering engine, which is the only one to support any top-layer elements besides fullscreen. In particular, it documents the current `<popup>` prototype, which is just one possibility for the general “popup” API being described here.

**<span style="text-decoration:underline;">Chromium 99.x behavior</span>**


<table>
  <tr>
   <td>
   </td>
   <td colspan="4" >Second Element
   </td>
  </tr>
  <tr>
   <td rowspan="4" > First Element</td>
   <td>
   </td>
   <td>Full screen
   </td>
   <td>Modal dialog
   </td>
   <td>`<popup>` element
   </td>
  </tr>
  <tr>
   <td>Full screen
   </td>
   <td>The second fullscreen element kicks the first one out of the top layer. ESC closes the second fullscreen element.
   </td>
   <td>Full screen stays visible, modal is displayed above fullscreen. ESC closes fullscreen <strong>first</strong>, second ESC closes dialog (<strong>backwards</strong>).
   </td>
   <td>Full screen stays visible, popup is displayed above fullscreen. ESC closes fullscreen <strong>first</strong>, second ESC closes popup (<strong>backwards</strong>).
   </td>
  </tr>
  <tr>
   <td>Modal dialog
   </td>
   <td>Full screen is displayed on top of modal, both are in the top layer. ESC closes full screen, second ESC closes dialog.
   </td>
   <td>Both dialogs are shown, first under second. First ESC closes second dialog, then first.
   </td>
   <td>Both are shown, dialog is under popup. Hitting ESC once closes <strong>both</strong> of them.
   </td>
  </tr>
  <tr>
   <td>`<popup>` element
   </td>
   <td>Full screen is displayed on top of popup, both are in the top layer. ESC closes full screen, second ESC closes popup.
   </td>
   <td>The dialog opening dismisses the popup. Hitting ESC closes the dialog.
   </td>
   <td>The second popup dismisses the first (assuming they’re not “ancestors” via DOM tree or anchor/popup attributes). Hitting ESC closes the popup.
   </td>
  </tr>
</table>


Note that the current fullscreen and `<dialog>` behavior does not prevent both types of elements from being in the top layer at once. In other words, there is currently no “one-at-a-time” managment of the top layer, other than the handling of the ESC key. In some cases the keyboard interaction pattern can be a bit confusing. For example, in several cases (the ones highlighted in red), hitting ESC first closes the element on the **bottom**, and a second ESC closes the element on the “top”. This is on purpose, for security reasons ([specified here](https://wicg.github.io/close-watcher/#close-signal-steps)): the ESC key must always close the fullscreen element and should not be cancellable.

Generally, [per spec](https://fullscreen.spec.whatwg.org/#new-stacking-layer), when there are multiple elements in the top layer, they are rendered in the order they were added to the top layer set. Each of these elements forms a stacking context, meaning z-index cannot be used to change this painting order.


## Other Top Layer Element Types

In addition to the element types described in the section above, there are a number of other UX patterns that deserve some attention. Bo Cupp put together [this list](https://docs.google.com/document/d/1I8Y7quk_Noim3GQ1ByhDWg1Jaj58OSE7wv2XUBHkkIU/edit#) containing some examples. It is interesting to examine a few of those, particularly Tooltip and Async, since those have different behavioral expectations about how they share the top layer.


### Tooltip

A tooltip is typically triggered/opened by hovering over an element for a prescribed delay period. It is dismissed by several things, including a pure delay, hitting ESC, moving the mouse outside the element, clicking on or focusing other elements (including the trigger element), etc. Tooltips cannot be focused and do not provide tab stops.

Importantly for this discussion, when a (top-layer) tooltip is shown, **other top-layer elements should** **remain visible**. This is because a tooltip is very transient UI that should not affect the remainder of the page state. Only one tooltip should be visible at a time: when one tooltip opens, all other tooltips should be closed. Nested tooltips (tooltips on tooltips) are technically possible but not common.


### Async

Async notifications (sometimes called “toasts”) are top-layer elements that are (typically) triggered asynchronously, such as via server-push or a timer. They may also be dismissed without user interaction, such as after a delay.

**Async notifications do not dismiss any other top layer elements.** They also do not dismiss each other, and instead “stack” somehow in the UI. They can be dismissed either explicitly by the user (one at a time, or all together), or by a timer.


# Shadow DOM

Some thought needs to be given to what happens if an element within a shadow root uses this API to move a shadow-DOM-contained element to the top layer. One use case of such an element would be a custom element that wraps a popup type UI element, such as a `<my-tooltip>`. Should it be “ok” for a shadow element (potentially inside even a closed shadow root) to be promoted to the top layer like this? Or should it be a requirement that the light-DOM element (e.g. the `<my-tooltip>` element) itself be the one to get promoted to the top layer?

Since the rendering output from a shadow host **is already** its shadow DOM content only, it would seem totally appropriate for shadow-contained elements to be allowed to move to the top layer, since the top layer is very similar to z-index and is just a layout convenience. It also seems considerably more ergonomic to allow this, rather than requiring the top-layer API be used at the highest light-DOM element containing the desired content. There are even cases where this wouldn’t make sense or be possible, such as a web component containing the entire page, `<my-app>`. In that case, it wouldn’t be possible for anything contained in the app to use the top layer API.

This should get additional brainstorming. 


# Exceeding the Frame Bounds

Allowing a popup/top-layer element to exceed the bounds of its containing frame poses a serious security risk: such an element could spoof browser UI or containing page content. While the [original `<popup>` proposal](https://open-ui.org/components/popup.research.explainer#goals) did not discuss this issue, the [`<selectmenu>` proposal](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ControlUICustomization/explainer.md#security) does have a specific section at least mentioning this issue. While it is possible to allow exceeding frame bounds in some cases, e.g. with the fullscreen API, great care must be taken in these cases to ensure user safety. It would seem difficult to allow an arbitrary element to exceed frame bounds using this (popup/top-layer) proposal. But perhaps there are safe ways to allow this behavior. More brainstorming is needed.

In the interim, two use counters were added to measure how often this type of behavior might be needed. They are approximations, as they merely measure the total number of times a “popup” is shown (including `<select>` popup, `<input type=color>` color picker, and `<input type=date/etc>` date/time picker), and the total number of times those popups exceed the owner frame bounds. Data can be found here:



* Total popups shown: [0.7% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3298)
* Popups appeared outside frame bounds: [0.08% of page loads](https://chromestatus.com/metrics/feature/timeline/popularity/3299)

So about 11% of all popups currently exceed their owner frame bounds. That should be considered a rough upper bound, as it is possible that some of those popups **could** have fit within their frame if an attempt was made to do so, but they just happened to exceed the bounds anyway.


# Eventual Single-Purpose Elements

There might come a time, sooner or later, where a new HTML element is desired which combines strong semantics and purpose-built behaviors. For example, a `<tooltip>` or `<listbox>` element. Those elements could be relatively easily built via the APIs proposed in this document. For example, a `<tooltip>` element could be defined to have role=tooltip, and could use the toplayer API for always-on-top rendering. It could define its default event handlers to perform light-dismiss and one-at-a-time management, via the same mechanisms described, e.g. [here](#bookmark=kix.v6oku3r2fwb4). In other words, these new elements could be explained in terms of the lower-level primitives being proposed for this API.
