# Resize Observer Explainer

This document outlines an API for observing DOM element's size.

## Why observe size changes?

Javascript is often used to create/position DOM children. Main reasons for doing this are:

* to load only a visible subset of all possible children.

* to responsively position children.

Both of these use cases are dependent upon the size of the parent element. The DOM manipulation code must know when parent's size changes, in order to redo its side-effects. There is no clean way to observe size changes today.

But there are workarounds.

## Current workarounds

Two main ways of detecting size changes are:

1) Size polling. This is an only option for component developers. Their components can be resized by hosts at any time. It is power-inefficient, janky, and a performance hit (querying element's size can trigger layout).

2) Ad-hoc notifications: Javascript that causes a resize is responsible for generating resize notifications. This solution is error-prone, cumbersome to maintain, and only works if developer is in control of the entire page.

> "We call notifyResize all the time" -- Polymer user

[Pattern samples](#common_practice) can be found at the bottom of this document.

None of existing workarounds are desirable. They fail in power consumption, code complexity, and perfomance.

## Goal

Create an API to notify developers when element's size changes. Notifications should be delivered when developers need them. Applications developed with the new API should be performant, and power efficient.

## Proposed API

The proposed API is an observer-style API. It is modeled after [other](https://www.w3.org/TR/dom/#mutation-observers) DOM [observers](https://github.com/WICG/IntersectionObserver/blob/master/explainer.md).

### Interface

    [Constructor(ResizeObserverCallback callback, optional ResizeObserverInit resizeObserverInitDict)]

    interface ResizeObserver {
        void observe(Element target);
        void unobserve(Element target);
        void disconnect();
    };

    dictionary ResizeObserverInit {
        ErrorCallback errorHandler;
    };

    callback ResizeObserverCallback = void(sequence<ResizeChangeRecord> changeset);

    interface ResizeChangeRecord {
        Element element;
        // TODO: is this the right information to pass back?
        double offsetWidth;
        double offsetHeight;
        double borderWidth;
        double borderHeight;
        double paddingWidth;
        double paddingHeight;
        boolean isAnimation;
    };

    callback ErrorCallback = void(DOMException error, sequence<ResizeChangeRecord> changeset);

### Design discussion

#### Why an observer based API, and not events?

Performance: resize notifications can be high frequency. Observer API avoids the significant overhead of events:

* event capture/bubble
* avoid calling O(n) callbacks for n elements.

Framework authors could provide a developer-friendly 'event-like' based API on top of ResizeObserver to avoid registering too many observers.

#### What triggers a resize notification?

ResizeObserver's clients are interested in manipulating content inside its content box. The minimal information they need is content box's size. This information can be computed with:

* offsetSize: css size + border + padding + scrollbar
* borderSize:
* paddingSize:

Changes in any of these should trigger a resize notification.

Edge case: what to do if changes are reverted before notification is fired? It is recommended that DOM can still generate a resize event. This avoids the complexity of having to keep track of the original element state.

#### What information do notifications contain?

The element.

Geometry information: offsetSize, borderSize, and paddingSize. Geometry is here to help authors avoid triggering layout.

isAnimationFlag???.

#### Inline elements

Inline elements should not generate resize notifications.

#### What about transforms?

Transforms do not affect offsetSize, borderSize, or paddingSize. They should not trigger notifications.

#### What about animations?

Animations that affect watched properties should trigger notifications.

Developers might want to skip doing work during animation if work is expensive. It might be useful to add an 'isAnimation' flag to make this easy.

#### Resizing and visibility

offsetSize becomes 0 when element is invisible. This will generate a resize notification. Developers will be able to use ResizeObserver to observe visibility.

#### When are notifications delivered?

Read [WebRenderingPipeline](http://bit.ly/rendering-pipeline) first to understand how the current rendering pipeline works.

The revised pipeline

![The revised pipeline](newpipeline.png)

The new resize notification callbacks fire once per frame after layout. The callbacks necessarily have to run between the Layout and Paint phases. If a layout is forced (e.g. during a setTimeout), then the callbacks do not fire until we run the whole pipeline before putting up a frame. So, for example, this means that a setTimeout will never be able to inject itself between the requestAnimationFrame and the resize observers.

The callbacks themselves can (and will) modify style and tree structure, so we need to run the whole Layout phase in a loop so that changes made during this frame actually get in before the paint. While looping is a little concerning, we already have this pattern in the platform (e.g. MutationObservers, Promises) and it hasn’t caused undue problems. The loop completes when the Notify step doesn’t dirty any style or layout state.

#### Dealing with infinite loops

The number of times notifications will run in a single frame has to be limited to prevent infinite loops. Error will be thrown if the number of loops exceeds the limit.

All registered error handlers will be notified of the error, not just the handlers for the existing changeset. We call all handlers because it is expected that most developers will not implement the error handler. The idea is that developers that really care will be able to detect errors caused by misbehaving components they are using.

The notifications not delivered because of the error will be delivered in the next delivery cycle. The API should never drop notifications. Missed notification would result in user-facing ugliness.

Looping number still needs to be determined empirically.

## Usage examples <a name="common_practice">


#### EXAMPLE 1: [Disqus](https://disqus.com/)

Discuss uses polling.

> The Disqus commenting system is embedded into other sites through an iframe.
When the content inside their iframe changes size they want to be able
to resize the top level iframe and move around elements inside
the iframe as well. Today they do this by periodically polling the
document height, through MutationObservers and requestAnimationFrame,
to detect when things might have changed. What they really want is a
resize listener instead so they can avoid polling and causing
unnecessary synchronous layouts. This has benefits for both battery
life and performance.

```html
    <!-- ResizeObserver works well for this use case -->
    <iframe id="toplevel">
        <div id="content"></div>
    </iframe>
    <script>
        var content = document.getElementById('content');

        var resizeObserver = new ResizeObserver(function handler(changes) {
            for (var i=0; i<changes.length; i++) {
                if (changes[i].element == content) {
                    console.log("resize the top level iframe and move around elements inside the iframe");
                }
            }
        });
        resizeObserver.observe( document.getElementById('content'));
    </script>
```

#### EXAMPLE 2: [Facebook](https://www.facebook.com/)

Facebook would like to use resize event to optimize friend list loading.

> use case we have right now is inserting a <div> that will hold a list of friends who are online in chat. As soon as we insert the <div> we want to know it's size so we can plan how many people we should render in the list. Querying the height right away triggers a layout, so we encourage people to do so inside a RAF. This requires cooperation between all RAF users to coordinate queries and modifications to the DOM. I think users of ResizeObserver would find a similar issue if the code triggered by one event modifies the DOM and the code from the next event queries it, a layout is forced.

```javascript
    // ResizeObserver helps: automatically places callback in RAF.
    // geometry information
    var friends = document.createElement('div');
    resizeObserver = new ResizeObserver(function handler(changes) {
        for (var i=0; i<changes.length;i++) {
            if (changes[i] == friends) {
                var howManyFriends = changes[i].geometry.height / 24;
            }
        }
    });
```

#### EXAMPLE 3: [Polymer.IronResizableBehavior](https://elements.polymer-project.org/elements/iron-resizable-behavior)

Polymer framework implements ad-hoc resize notification

> IronResizableBehavior is a behavior that can be used in Polymer elements to coordinate the flow of resize events between "resizers" (elements that control the size or hidden state of their children) and "resizables" (elements that need to be notified when they are resized or un-hidden by their parents in order to take action on their new measurements).

The entire [Polymer.IronResizableBehavior](https://github.com/PolymerElements/iron-resizable-behavior/blob/master/iron-resizable-behavior.html) can be reimplemented with ResizeObserver.


#### EXAMPLE 4: [Google Maps API](https://developers.google.com/maps/documentation/javascript/3.exp/reference)

Google Maps API requires developers to notify map when it has been resized

> `resize` event:  Developers should trigger this event on the map when the div changes size: google.maps.event.trigger(map, 'resize') .

Resize event can be hidden

#### EXAMPLE 5: An infinite scroller

TODO

### Comic relief

There exists a [very creative solution](http://www.backalleycoder.com/2013/03/18/cross-browser-event-based-element-resize-detection/) to resize detection today. It is cross-browser, and based upon existing standards. To track elements size, create a child iframe that is absolutelly pinned to element's size. iframe is a window, so it gets window.resize event. The penalty is the iframe creation cost, 0.5Mb per event listener.



