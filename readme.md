# ModernHistory.js

[ModernHistory.js](http://javajosh.com/ModernHistory.js/) is a javascript library to smooth over a tricky part of the browser history API
using minimal, readable, modern JavaScript (ES6) that has no dependencies. A non-goal is to provide a
library that can provide history services for older browsers (a polyfill). If you want a polyfill,
take a look at [history.js](https://github.com/browserstate/history.js).

ModernHistory deals correctly with hash clicks (anchor links), pushState, and browser refresh.
See below for a demo.

## Usage
Your code uses ModernHistory.js like this:

```js
    <script src="ModernHistory.js"></script>
    <script>
        ModernHistory.debug = false; //set to true to get output similar to the following

        // Instead of window.onpopstate, register for these custom events.
        window.addEventListener('ModernHistory.add', e =>{
            console.log('add', ModernHistory.state, ModernHistory.state.detail);
        });
        window.addEventListener('ModernHistory.move', e =>{
            console.log('mov', ModernHistory.state);
        });
        window.addEventListener('ModernHistory.forward', e =>{
            console.log('for', ModernHistory.state);
        });
        window.addEventListener('ModernHistory.back', e =>{
            console.log('bak', ModernHistory.state);
        });
        window.addEventListener('ModernHistory.recoverState', e =>{
            console.log('rec', ModernHistory.state);
        });
        // Instead of
        ModernHistory.addState('a', {a:1, b:2});
        ModernHistory.addState('b');
    </script>
```


There are a lot of subtle issues (covered in [log](/log)), but the basic idea is to *normalize* all the ways programs
can interact with history, and offer a better *representation* of history state. The representation
is called `ModernHistory` and it is a global object you can inspect directly, and it might
look something like this:

```js
ModernHistory = {
    "addState": label => {}      // Call this function instead of window.history.pushState
    "debug" : false              // A (writable) flag that can enable debug output to console
    "R": 2,                      // Number of resources accessed before this SPA
    "dir" : 1,                   // The direction of the last movement. (-1, 0, 1) = (back, none, forward)
    "pos" : 4,                   // Where we currently are in the timeline.
    "state" : {...},             // The current state; = window.history.state = timeLine(pos)
    "timeLine" : [state,..],     // All states accessible in this tab
    "isRefresh" : false          // If the tab is refreshed, then there may be future states we don't know about yet.
}
```


Each state object looks something like this:

```js
state = {
    "id": 1,                     // Equal to it's index in timeLine
    "label": "a",                // Every state has a simple text label
    "timestamp": 23423,          // When the browser created this state.
    "isHash": false,             // True if this state was triggered by a hash change.
    "detail": {...},             // Your application specific state, or null
    "prevState": {...},          // Reference to previous and last states
    "nextState": {...}
}
```


### Limitations
Unfortunately, there is no way to allow application code to call `window.history.pushState()`
directly. This is because there is no `onpushstate` event defined in the DOM.
(There may be a way by replacing the function outright, but I am hesitant to modify standard DOM objects
like this. Another option is to write a Proxy around the function, which is something I may explore later.)

Prefer to use the ModernHistory events rather than onpopstate, since onpopstate
is a much lower-level event, and can mean any of: forward, back, or adding a hash click. ModernHistory
differentiates between these cases, and provides a context for understanding what's going on.

If your code calls `window.pushState()` ModernHistory.js will fail.

### Ideas!
  1. Keep track of a tree of history.
  2. Wrap `pushState()` in a proxy to create `onpushstate`
  3. Investigate how and why the browser removes references to future history after refresh.

## Demo

GitHub Markdown strips JavaScript, so load the demo by going to the 
[ModernHistory.js Homepage](http://javajosh.com/ModernHistory.js/)

## Known Issues

If you build some state, go back, and refresh the page, the timeLine will be truncated at the state you went back to.
For reasons that are mysterious to me, the "nextState" element of the current state is removed on refresh. I even
attempted to add a reference to the timeLine to each event, and this reference was removed on refresh. This is some
serious voodoo magic! I'd also like to detect pushState with a proxy, and the script itself could probably use some
packaging idiom love from the community.

Truth is this script was a lot simpler before I realized I needed to handle <i>browser refresh</i> . Refresh nukes
all program state, which in this case means the `timeLine`, but the browser tab still
maintains history state knows about forward and back history; a somewhat elegant solution
to this inelegant problem was to form a doubly-linked list between states, and walk through them all
on page load, and rebuild the timeLine. However, this mysteriously failed, because all references to future
states were removed during the refresh. It doesn't seem like a logic error on my part, but rather some
fancy browser engine thing that hides information from the page. After exploring some possibilities,
I've settled on grabbing all past states and recovering future states as the user visits them again.
(This is why `isRefresh=true` means that the `timeLine` may be incomplete, and in fact
can grow even without you calling `addState()`!

*Josh Rehman, Oct 2017*
