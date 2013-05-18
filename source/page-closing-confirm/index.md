# Page Closing Confirmation with jQuery Custom Event

- tags: jQuery, event, beforeunload
- pubdate: 2013-05-06

---

I came across a problem when building a UCG website: how to issue a warning when the user is leaving the current page but has unsaved content?

The answer is actually quite simple. A quick search would lead you to the [`beforeunload`][mdn-beforeunload] event, which is triggered when a document is *about to* be unloaded. Speaking less technically, `beforeunload` is triggered when a the page is about to close, e.g. the user closes the page, refreshes the page or goes to another page (by entering a new URL in the address bar or following a hyperlink). Whatever, this is just what we need. BTW, hooking [`unload`][mdn-unload] is too late.

[mdn-beforeunload]: https://developer.mozilla.org/en-US/docs/DOM/Mozilla_event_reference/beforeunload
[mdn-unload]: https://developer.mozilla.org/en-US/docs/DOM/Mozilla_event_reference/unload

According to the documentation on MDN:

> When a non-empty string is assigned to the returnValue Event property, a dialog box appears, asking the users for confirmation to leave the page (see example below). When no value is provided, the event is processed silently.

Now you could have something like the boilerplate code:

```javascript
$(window).on('beforeunload', function(e) {
    if(hasUnsaved()) {
        return 'You have unsaved stuff. Are you sure to leave?';
    }
});
```

With this piece of code, the user will get a confirmation box under the situation described above. Yikes! You should also note that as the documentation states, the only requirement to trigger the confirmation process is to return an non-empty string and the browser is not obliged to display your message to the user. (On my machine, Firefox and Safari ignore it, while Chrome displays it.)

So far, so great. But our situation is a bit more complex than that: as users can input multiple types of content, say text, picture and video, each type of media is handled in separate javascript files. (Considering that we may have even more types of input, modularization is the sane way to go.) So when the user closes the page, the following things should be checked:

* has the user inputed any text?
* has the user uploaded some picture?
* has the user included a video link from youtube?
* …and so on

Putting all these in one grand event handler is a rather bad idea as this breaks modularization. Letting each module handle `beforeunload` respectively seems better but the **intention** would be broken to several places. What can we do?

Fortunately we can achieve this with custom jQuery event. Let the event type be `webapp:page:closing`. We define a protocol over it:

1. When the page is about to unload, `webapp:page:closing` is triggered and we *assume* that the default behavior of the event is to close the page without any confirmation.
2. If `preventDefault` of the event object is called, the default behavior is cancelled, i.e. the user has to confirm closing page.
3. A module can optionally attach a message for the user, but like `beforeunload`, the message may be ignored by the browser.

With reference to the [Event Object][jquery-event-object] documentation and [this SO answer][so-event-custom-value] concerning custom property of event, we turn the boilerplate code above into this (e.g., in `core.js`):

[jquery-event-object]: http://api.jquery.com/category/events/event-object/
[so-event-custom-value]: http://stackoverflow.com/a/11077126/1240620

```javascript
$(window).on('beforeunload', function() {
    var e = $.Event('webapp:page:closing');
    $(window).trigger(e); // let other modules determine whether to prevent closing
    if(e.isDefaultPrevented()) {
        // e.message is optional
        return e.message || 'You have unsaved stuff. Are you sure to leave?';
    }
});
```

and in `module-text-input.js`, the related code would be like:

```javascript
$(window).on('webapp:page:closing', function(e) {
    if(textAlreadyInputed()) {
        e.preventDefault();
        e.message = 'You have already inputed some text. Sure to leave?';
    }
});
```

You can see that `webapp:page:closing` event is something like `beforeunload` yet with a more regular interface (cancellable with `preventDefault`). This solution has all the benefits we want: clear intention, keeping modularization as well as decentralized control.

Hope this post gives you some insight on the usage of jQuery custom event!

Note: the above code is tested on Firefox 20.0, Chrome 26.0 and Safari 6.0 under OS X ML. For compatibility issue concerning `beforeunload`, you can refer to [this post][beforeunload-compatibility].

[beforeunload-compatibility]: http://jonathonhill.net/2011-03-04/catching-the-javascript-beforeunload-event-the-cross-browser-way/
