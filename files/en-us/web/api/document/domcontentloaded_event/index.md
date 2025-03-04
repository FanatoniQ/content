---
title: 'Document: DOMContentLoaded event'
slug: Web/API/Document/DOMContentLoaded_event
page-type: web-api-event
tags:
  - API
  - DOMContentLoaded
  - Document
  - Event
  - Web
browser-compat: api.Document.DOMContentLoaded_event
---
{{APIRef}}

The **`DOMContentLoaded`** event fires when the initial HTML document has been completely loaded and parsed, without waiting for stylesheets, images, and subframes to finish loading.

<table class="properties">
  <tbody>
    <tr>
      <th scope="row">Bubbles</th>
      <td>Yes</td>
    </tr>
    <tr>
      <th scope="row">Cancelable</th>
      <td>No</td>
    </tr>
    <tr>
      <th scope="row">Interface</th>
      <td>{{domxref("Event")}}</td>
    </tr>
    <tr>
      <th scope="row">Event handler property</th>
      <td>None</td>
    </tr>
  </tbody>
</table>

A different event, {{domxref("Window/load_event", "load")}}, should be used only to detect a fully-loaded page. It is a common mistake to use `load` where `DOMContentLoaded` would be more appropriate.

Synchronous JavaScript pauses parsing of the DOM. If you want the DOM to get parsed as fast as possible after the user has requested the page, you can make your [JavaScript asynchronous](/en-US/docs/Web/API/XMLHttpRequest/Synchronous_and_Asynchronous_Requests) and [optimize loading of stylesheets](https://developers.google.com/speed/docs/insights/OptimizeCSSDelivery). If loaded as usual, stylesheets slow down DOM parsing as they're loaded in parallel, "stealing" traffic from the main HTML document.

## Polyfill

The following code ports the functionality of the `DOMContentLoaded` event all the way back to IE6+, with a fallback to `window.onload` that works everywhere.

```js
// Usage: DOMContentLoaded(function(e) { console.log(e); /* your code here */});

function DOMContentLoaded() { "use strict";

    var ael = 'addEventListener', rel = 'removeEventListener', aev = 'attachEvent', dev = 'detachEvent';
    var alreadyRun = false,
        funcs = arguments; // for use in the idempotent function `ready()`, defined later.

    function microtime() { return + new Date() } // new Date().valueOf();

    /* The vast majority of browsers currently in use now support both addEventListener
       and DOMContentLoaded. However, 2% is still a significant portion of the web, and
       graceful degradation is still the best design approach.

       `document.readyState === 'complete'` is functionally equivalent to `window.onload`.
       The events fire within a few tenths of a second, and reported correctly in every
       browser that was tested, including IE6. But IE6 to 10 did not correctly return the other
       readyState values as per the spec:
       In IE6-10, readyState was sometimes 'interactive', even when the DOM wasn't accessible,
       so it's safe to assume that listening to the `onreadystatechange` event
       in legacy browsers is unstable. Should readyState be undefined, accessing undefined properties
       of a defined object (document) will not throw.

       The following statement checks for IE < 11 via conditional compilation.
       `@_jscript_version` is a special String variable defined only in IE conditional comments,
       which themselves only appear as regular comments to other browsers.
       Browsers not named IE interpret the following code as
       `Number( new Function("")() )` => `Number(undefined)` => `NaN`.
       `NaN` is neither >, <, nor = to any other value.
       Values: IE5: 5?, IE5.5: 5.5?, IE6/7: 5.6/5.7, IE8: 5.8, IE9: 9, IE10: 10,
       (IE11 older doc mode*): 11, IE11 / NOT IE: undefined
    */

    var jscript_version = Number( new Function("/*@cc_on return @_jscript_version; @*\/")() );

    // check if the DOM has already loaded
    // If it has, send null as the readyTime, since we don't know when the DOM became ready.

    if (document.readyState === 'complete') { ready(null); return; } // execute ready()

    // For IE<9 poll document.documentElement.doScroll(), no further actions are needed.
    if (jscript_version < 9) { doIEScrollCheck(); return; }

    // ael: addEventListener, rel: removeEventListener, aev: attachEvent, dev: detachEvent

    if (document[ael]) {
        document[ael]("DOMContentLoaded", ready, false);

        // fallback to the universal load event in case DOMContentLoaded isn't supported.
        window[ael]("load", ready, false);
    } else
    if (aev in window) { window[aev]('onload', ready);
        // Old Opera has a default of window.attachEvent being falsy, so we use the in operator instead.
        // https://dev.opera.com/blog/window-event-attachevent-detachevent-script-onreadystatechange/
    } else {
        // fallback to window.onload that will always work.
        addOnload(ready);
    }

    // addOnload: Allows us to preserve any original `window.onload` handlers,
    // in ancient (prehistoric?) browsers where this is even necessary, while providing the
    // option to chain onloads, and dequeue them later.

    function addOnload(fn) {

        var prev = window.onload; // old `window.onload`, which could be set by this function, or elsewhere.

        // Here we add a function queue list to allow for dequeueing.
        // Should we have to use window.onload, `addOnload.queue` is the queue of functions
        // that we will run when the DOM is ready.

        if ( typeof addOnload.queue !== 'object') { // allow loose comparison of arrays
            addOnload.queue = [];
            if (typeof prev === 'function') {
                addOnload.queue.push( prev ); // add the previously defined event handler, if any.
            }
        }

        if (typeof fn === 'function') { addOnload.queue.push(fn) } // add the new function

        window.onload = function() { // iterate through the queued functions
            for (var i = 0; i < addOnload.queue.length; i++) { addOnload.queue[i]() }
        };
    }

    // dequeueOnload: remove a queued `addOnload` function from the chain.

    function dequeueOnload(fn, all) {

        // Sort backwards through the queued functions in `addOnload.queue` (if it's defined)
        // until we find `fn`, and then remove `fn` from its place in the array.

        if (typeof addOnload.queue === 'object') { // array
            for (var i = addOnload.queue.length-1; i >= 0; i--) { // iterate backwards
                if (fn === addOnload.queue[i]) {
                    addOnload.queue.splice(i,1); if (!all) {break}
                }
            }
        }
    }

    // ready: idempotent event handler function

    function ready(ev) {
        if (alreadyRun) {return} alreadyRun = true;

        // This time is when the DOM has loaded, or, if all else fails,
        // when it was actually possible to inference that the DOM has loaded via a 'load' event.

        var readyTime = microtime();

        detach(); // detach any event handlers

        // run the functions (`funcs` is arguments of DOMContentLoaded)
        for (var i=0; i < funcs.length; i++) {

            var func = funcs[i];

            if (typeof func === 'function') {

                // force set `this` to `document`, for consistency.
                func.call(document, {
                  'readyTime': (ev === null ? null : readyTime),
                  'funcExecuteTime': microtime(),
                  'currentFunction': func
                });
            }
        }
    }

    // detach: detach all the currently registered events.

    function detach() {
        if (document[rel]) {
            document[rel]("DOMContentLoaded", ready); window[rel]("load", ready);
        } else
        if (dev in window) { window[dev]("onload", ready); }
        else {
            dequeueOnload(ready);
        }
    }

    // doIEScrollCheck: poll document.documentElement.doScroll until it no longer throws.

    function doIEScrollCheck() { // for use in IE < 9 only.
        if ( window.frameElement ) {
            /* We're in an `iframe` or similar.
               The `document.documentElement.doScroll` technique does not work if we're not
               at the top-level (parent document).
               Attach to onload if we're in an <iframe> in IE as there's no way to tell otherwise
            */
            try { window.attachEvent("onload", ready); } catch (e) { }
            return;
        }
        // if we get here, we're not in an `iframe`.
        try {
            // when this statement no longer throws, the DOM is accessible in old IE
            document.documentElement.doScroll('left');
        } catch(error) {
            setTimeout(function() {
                (document.readyState === 'complete') ? ready() : doIEScrollCheck();
            }, 50);
            return;
        }
        ready();
    }
}

// Tested via BrowserStack.
```

## Examples

### Basic usage

```js
document.addEventListener('DOMContentLoaded', (event) => {
    console.log('DOM fully loaded and parsed');
});
```

### Delaying DOMContentLoaded

```html
<script>
  document.addEventListener('DOMContentLoaded', (event) => {
    console.log('DOM fully loaded and parsed');
  });

for( let i = 0; i < 1000000000; i++)
{} // This synchronous script is going to delay parsing of the DOM,
   // so the DOMContentLoaded event is going to launch later.
</script>
```

### Checking whether loading is already complete

`DOMContentLoaded` may fire before your script has a chance to run, so it is wise to check before adding a listener.

```js
function doSomething() {
  console.info('DOM loaded');
}

if (document.readyState === 'loading') {  // Loading hasn't finished yet
  document.addEventListener('DOMContentLoaded', doSomething);
} else {  // `DOMContentLoaded` has already fired
  doSomething();
}
```

### Live example

#### HTML

```html
<div class="controls">
  <button id="reload" type="button">Reload</button>
</div>

<div class="event-log">
  <label for="eventLog">Event log:</label>
  <textarea readonly class="event-log-contents" rows="8" cols="30" id="leventLog"></textarea>
</div>
```

```css hidden
body {
  display: grid;
  grid-template-areas: "control log";
}

.controls {
  grid-area: control;
  display: flex;
  align-items: center;
  justify-content: center;
}

.event-log {
  grid-area: log;
}

.event-log-contents {
  resize: none;
}

label, button {
  display: block;
}

#reload {
  height: 2rem;
}
```

#### JS

```js
const log = document.querySelector('.event-log-contents');
const reload = document.querySelector('#reload');

reload.addEventListener('click', () => {
  log.textContent ='';
  window.setTimeout(() => {
      window.location.reload(true);
  }, 200);
});

window.addEventListener('load', (event) => {
    log.textContent += 'load\n';
});

document.addEventListener('readystatechange', (event) => {
    log.textContent += `readystate: ${document.readyState}\n`;
});

document.addEventListener('DOMContentLoaded', (event) => {
    log.textContent += 'DOMContentLoaded\n';
});
```

#### Result

{{ EmbedLiveSample('Live_example', '100%', '160px') }}

## Specifications

{{Specifications}}

## Browser compatibility

{{Compat}}

## See also

- Related events: {{domxref("Window/load_event", "load")}}, {{domxref("Document/readystatechange_event", "readystatechange")}}, {{domxref("Window/beforeunload_event", "beforeunload")}}, {{domxref("Window/unload_event", "unload")}}
- This event on {{domxref("Window")}} targets: {{domxref("Window/DOMContentLoaded_event", "DOMContentLoaded")}}
