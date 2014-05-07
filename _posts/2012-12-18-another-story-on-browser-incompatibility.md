---
layout: post
title: "Another Story on Browser Incompatibility"
date: 2012-12-18
tags: browser javascript css dom standard incompatible jQuery
description: "Github's then textarea mirror function had a cross-browser issue. This article explains why this happens and how to fix it."
---

It has been months since the last post of this blog was published.
Actually, I am working as intern at a startup since July and have been busy building a website.

Each modern website is embracing social features and the one I am up to is no exception.
And sure, people use @john\_doe to mention John Doe at a social website.
So @-mention is an essential feature a social website is supposed to support.
But...

To get that @-mention work, I took some time to investigate several existing implementations.
(And you bet, to draw some inspirations from them, certainly!)
It seems that there are four major ways to implement this feature:

1. [twitter](http://twitter.com)'s way: does not permit duplicate username, use @john\_doe directly
2. [Renren](http://www.renren.com)'s way: permit duplicate username, use plain-text input, append user ID to user name, e.g. @John Doe(20121221)
3. [Facebook](http://www.facebook.com)'s way: permit duplicate, use plain-text input, mark mentioned user's name with background
4. [Google+](http://plus.google.com)'s way: permit duplicate, use rich-text input (with `contenteditable=true`), use `<input type=button>` to represent mentioned user.
Put user's name in `value` attribute and user's ID in `data-ID` attribute.

Obviously, the robustness rises from the first to the last, just as the level of difficulty to implement does.
(BTW, I have to put a special awe to Google+'s genius implementation.)
And it seems to implement this feature, there are [plenty][1] of [problem][5] to solve, and even more if [you][4] [use][3] [contenteditable][2] (just as Google+ does).
Since we favor an iterative approach and want to have a prototype first, I'll stick with `<textarea>`.

Enough for the background.
Now the problem becomes, *in order to popup a floating menu for user to select mentioned user, how to determine the position of current cursor in the `<textarea>`?*
The best solution I could find so far is [this][5].
And I was even lucky to find one such function from one of [github](http://github.com)'s compressed js file.
A little decoding of the compressed source reveals the implementation as follows
(a jQuery plugin as you can tell):

```javascript
(function(window, undefined) {
    var $ = window.jQuery;
    var document = window.document;

    var fixedStyles = [
        'position: absolute;',
        'overflow: auto;',
        'white-space: pre-wrap;',
        'word-wrap: break-word;',
        'top: 0px;',
        'left: 0px;'
    ];

    var variableStyles = ['box-sizing', 'font-family', 'font-size', 'font-style',
        'font-variant', 'font-weight', 'height', 'letter-spacing', 'line-height',
        'padding', 'text-decoration', 'text-indent', 'text-transform', 'width',
        'word-spacing'];

    $.fn.textareaMirror = function(pos) {
        // make sure we are manipulating a <textarea>
        if(! this[0])
            return;
        var ta = this[0];
        if(ta.nodeName.toLowerCase() !== 'textarea')
            return;

        // create mirror <div> right next to the <textarea>
        var mirror = $(ta).next()[0];
        if(mirror && $(mirror).is('div.js-textarea-mirror')) {
            mirror.innerHTML = '';
        } else {
            mirror = document.createElement('div');
            mirror.className = 'js-textarea-mirror';
        }

        // retrieve and apply styles of <textarea> to mirror <div>
        var fabricatedStyles = fixedStyles.slice(0);
        var computedStyles = window.getComputedStyle(ta);
        for(var i = 0;i < variableStyles.length;i++) {
            var property = variableStyles[i];
            fabricatedStyles.push('' + property + ':' + computedStyles[property] + ';');
        }
        mirror.style.cssText = fabricatedStyles.join('');

        var text, textNode1, textNode2;
        if(pos) {
            if(text = ta.value.substring(0, pos))
                textNode1 = document.createTextNode(text);
            if(text = ta.value.substring(pos))
                textNode2 = document.createTextNode(text);
        } else if(text = ta.value) {
            textNode1 = document.createTextNode(text);
        }

        // create marker
        var marker = document.createElement('span');
        marker.className = 'js-marker';
        marker.innerHTML = '&nbsp;';

        // append marker and text nodes to mirror
        if(textNode1)
            mirror.appendChild(textNode1);
        mirror.appendChild(marker);
        if(textNode2)
            mirror.appendChild(textNode2);

        if(! mirror.parentElement)
            // set mirror next to <textarea> if it is newly created
            $(ta).after(mirror);

        // set the same scroll offset
        mirror.scrollTop = ta.scrollTop;

        return mirror;
    };
})(window);
```

The source should be understood without much difficulty with the Stackoverflow answer above and the annotations.
The `pos` parameter, which needs a bit explanation, handles the situation when the cursor in the middle of the text.

However, the so-called textarea mirror does not work quite as expected in Firefox, my major browser.
Interestingly, in Chrome, it works perfectly.
See the picture below.

![](/assets/img/textarea-mirror-firefox-vs-chrome.png)

(The black text is the actual inputted text in textarea while the red one is that in the "mirror".
The blue line, if you are sharp to notice it, is the marker `<span>`.)

As you can tell. The effect in Firefox is totally undesirable. So why?
After some debugging, I nailed down the cause at line 41, where `computedStyles[property]` is used.
In that statement, a CSS property like `text-decoration` is used to retrieve the value, which works perfectly in Chrome but returns `undefined` in Firefox.
And with a little experiment, it seems that using property name like `textDecoration` (just what you'd write to access value in `element.style`) works in both browsers.
So after adding the following function

```javascript
function cssPropertyToJS(prop) {
    return prop.split('-').map(function(w, i) {
        return i==0 ? w : w[0].toUpperCase() + w.slice(1);
    }).join('');
}
```

and changing the former line 41 to

```javascript
fabricatedStyles.push('' + property + ':' + computedStyles[cssPropertyToJS(property)] + ';');
```

everything now works nicely. Yepp.

**An incompatibility between Firefox and Chrome!**

Just out of curiosity, I wonder: who's doing it wrong? Firefox or Chrome?
Documentation at [MDN][6] tells you that `window.getComputedStyle` returns `CSSStyleDeclaration`, which is defined in [DOM Level 2 Style][7].
Now it turns out that, with `CSSStyleDeclaration` you can only access property value via `getPropertyValue` and no attribute like `text-decoration` or `textDecoration` is defined in `CSSStyleDeclaration`.

WTH? We are at the end accessing some non-standard property?
Lucily not.
At the [end][8] of the same documentation, we can find the `CSS2Properties` extended interface, where all those `backgroundPosition` or `listStylePosition` stuff are defined. Aha!
So the conclusion goes that both browser implemented the essential as well as the extended interface. And diligent Chrome (or Webkit?) developers added CSS property support.

Now that the root cause is clear, the main story can end.
Here's a dessert if you are still interested in the correction to that textarea mirror plugin:
the `cssPropertyToJS` function seems **far from** optimal, so can we do better?
After some pondering, I bet there must be similar stuff in jQuery so I decided to refer to jQuery's implementation.
And [here][9] it is:

```javascript
var rdashAlpha = /-([a-z])/ig,
    fcamelCase = function( all, letter ) {
        return letter.toUpperCase();
    };
// usage
var name = 'font-family';
name = name.replace(rdashAlpha, fcamelCase);
```
What a clean, simple and efficient implementation!
This again reminds me of the power of regular expression.

A last joke: I haven't tested that on IE yet.

---

Update: I found [AT.js](https://github.com/ichord/At.js), an @-mention implementation using similar techniques as described above, only more robust and with more features.
I've already switched to it and hope you give it a try.

[1]: http://stackoverflow.com/questions/7497824/how-to-highlight-friends-name-in-facebook-status-update-box-textarea
[2]: http://stackoverflow.com/questions/1181700/set-cursor-position-on-contenteditable-div
[3]: http://stackoverflow.com/questions/2903991/how-to-detect-ctrlv-ctrlc-using-javascript
[4]: http://stackoverflow.com/questions/6022551/pasting-into-contentedittable-results-in-random-tag-insertion
[5]: http://stackoverflow.com/questions/3510009/textarea-caret-coordinates-jquery
[6]: https://developer.mozilla.org/en-US/docs/DOM/window.getComputedStyle
[7]: http://www.w3.org/TR/DOM-Level-2-Style/css.html#CSS-CSSStyleDeclaration
[8]: http://www.w3.org/TR/DOM-Level-2-Style/css.html#CSS-CSS2Properties
[9]: https://github.com/jquery/jquery/blob/master/speed/jquery-basis.js#L4545
