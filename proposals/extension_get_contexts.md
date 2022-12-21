# New API: extension.getContexts()

## Background / Summary

Chromium currently has the
[`extension.getViews()`](https://developer.chrome.com/docs/extensions/reference/extension/#method-getViews)
API method which allows an extension to get information about the views the
extension is running in. Common reasons for calling this method are: determining
if a toolbar popup is open, reading and/or modifying a settings page, and etc.
Chromium's implementation of Manifest V2 (MV2) allows for
[background scripts (pages)](https://developer.chrome.com/docs/extensions/mv2/background_pages/)
to call this API. This is possible because these scripts run on our main thread
which allows them easy access to the JavaScript
[`Window`](http://go/mdn/API/Window#instance_properties) objects provided by
`extension.getViews()`.

## Problem

However, with Chromium’s migration to Manifest V3 (MV3), background pages
[no longer exist](https://developer.chrome.com/docs/extensions/mv3/migrating_to_service_workers/);
instead, an extension’s background context is
[service worker](https://developer.chrome.com/docs/workbox/service-worker-overview/)-based.
For technical reasons<sup>[1](#footnotes)</sup>, service workers cannot access
the [`Window`](http://go/mdn/API/Window#instance_properties) objects that
`extension.getViews()` provides and it is not feasible to implement that with
our browser design. Due to this we cannot provide access to the JavaScript
context for these views, but we can allow an extension to query for them and
potentially target them for messaging purposes.

## Solution

Considering the above situation, we’d like to propose a new `extension` API
method to asynchronously provide metadata about the view that is still useful
for an extension: `extension.getContexts()`. This will allow extension background scripts to identify the active contexts in the extension.
For example, this can be used to target messages to send using
[`runtime.sendMessage()`](https://developer.chrome.com/docs/extensions/reference/runtime/#method-sendMessage),
etc.). Introducing this API will allow an easier migration from MV2.

## API Proposal

This new API would use a similar parameter signature as
`extension.getViews()`but instead accept a new `contextProperties` object.

```
extension.getContexts()(
  contextProperties?: object,
)
```

`contextProperties` will have similar behavior to `extension.getViews()`’s
[`fetchProperties`](https://developer.chrome.com/docs/extensions/reference/extension/#method-getViews)
object in that it will limit the scope of the query to certain tabs, context
types, and windows. It would be defined as:

```
extension.ContextProperties = {
  tabId? = int,                // Find context by tab id, omitted returns all tabs.
  contextType? = ContextType,  // Find context by type, omitted returns all types.
  windowId? = int,             // Find context by window id, omitted returns all
                               // windows.
}
```

`ContextType` would serve a similar purpose of
[`fetchProperties`](https://developer.chrome.com/docs/extensions/reference/extension/#method-getViews)’s
[`ViewType`](https://developer.chrome.com/docs/extensions/reference/extension/#type-ViewType)
in order to limit the type of extension contexts returned by the API. It would
be defined as:

```
extension.ContextType = {
  TAB: 'TAB',                               // Tabs the extension is running in.
  POPUP: 'POPUP',                           // Popups created by the extension.
  BACKGROUND: 'BACKGROUND',                 // Background script for the extension.
                                            // (background pages for MV2, or service
                                            // workers for MV2 + MV3).
  OFFSCREEN_DOCUMENT: 'OFFSCREEN_DOCUMENT'  // Offscreen documents for the extension.
};
```

It would return an array of new `Context` objects for each frame the extension
is running in:

```
extension.Context = {
  tabId: int,                    // ID of the tab for this view.
  windowId: int,                 // ID of the window this tab is located in.
  documentId: int,               // ID of the DOM document loaded in the tab.
  frameId: int,                  // ID of the frame this context is running for.
  contextType: ContextType,      // Context type -- tab, popup, etc.
  frameUrl: string,              // The current URL of the frame. For background
                                 // pages and service workers this value is empty.
  origin: string,                // The origin of the context. Mostly this is the
                                 // extension origin.
  incognito: boolean             // Whether the context is for an incognito
                                 // profile.
}
```

`Context` objects will contain an `incognito` value indicating if the Context is
running in an incognito profile. In this area, this brings up the case of an
extension specifying
[split incognito mode](https://developer.chrome.com/docs/extensions/mv3/manifest/incognito/#split).
In this case we would not return the `Context`s from the related (but different)
extension process running for the incognito profile. `spanning`mode would return
any `Context`s running in an incognito profile (with `Context.incognito` being
`true`) in the same shared process. More comments about this are in the Future
Work](#future-work) section on this proposal.

## Future Work

### Multi Page Architecture Fields (`DocumentLifeCycle`, `FrameType`, and `parentDocumentId`, and `parentFrameId`)

[Multi Page Architecture](https://docs.google.com//1NginQ8k0w3znuwTiJ5qjYmBKgZDekvEPC22q0I4swxQ#heading=h.w1qo2n6sr8wn)
caused
[multiple changes](https://developer.chrome.com/blog/extension-instantnav/) to
Chromium's
[tabs API](https://developer.chrome.com/docs/extensions/reference/tabs/),
[scripting API](https://developer.chrome.com/docs/extensions/reference/scripting/),
and
[web navigation API](https://developer.chrome.com/docs/extensions/reference/webNavigation/).
For the web navigation API it changed how outermost frames are exposed to
extensions in Chromium. Considering this new architecture it is likely that
extensions will want to know what navigation state (`DocumentLifeCycle`) a
`Context` is in, whether the frame’s `FrameType` is an outermost frame, and
other fields like `parentDocumentId`, and `parentFrameId`. These are fields we
would like to include in a future iteration of this API.

### Asynchronous and Synchronous Version

A potential avenue is that, since this new API is asynchronous whereas
`getViews()` is synchronous, we could explore a synchronous version of this new
API if there’s significant demand for it. That demand would need to justify why
the usage of `await` on the existing asynchronous API isn't sufficient. As
mentioned previously<sup>[1](#footnotes)</sup>, there are technical limitations
that make it infeasible to synchronously fetch all `Context`s. We could provide
two methods, one synchronous, and one asynchronous API methods:

*   `getContexts()`: as defined above and asynchronous. It fetches all
    `Context`s available in the process running for the profile.
*   `getContextsSynchronous()` (name TBD): defined similarly to `getContexts()`,
    but is synchronous. It returns the data for other `Context`s that are
    available synchronously (running in the same process and on the same
    thread). If called from a background script, it would likely only return the
    background script `Context`.

### Content Script Contexts

[Content scripts](https://developer.chrome.com/docs/extensions/mv3/content_scripts/)
run in a separate
[Renderer](https://developer.chrome.com/blog/inside-browser-part3/) (and
process) from the extension process. This proposal would not return those
`Context`s since this adds significant complexity to the design in querying all
Renderer processes in the browser to find content scripts. However, extensions
could find it useful to have this `Context` metadata for places where the
extension is executing JavaScript in a web page. It could be justified in the
future if extension developers would have a significant need. If implemented the
`frameUrl` field would be empty as providing a URL to the content script (within
the extension) doesn't have a use case.

### Messaging APIs Support `Context`s as Targets

This new API now provides the ability to consider specifying targets via
`Context`s to messaging APIs like `runtime.sendMessage()` or
`tabs.sendMessage()`. With this support it would avoid
[various workarounds](https://github.com/w3c/webextensions/pull/334#issuecomment-1343255899)
needed to target specific pages with messages. `tabs.sendMessage()` accepting
`Context`s would require work in retrieving content script `Context`s explained
[above](#content-script-contexts).

## Open Questions

### Include `sandbox` Pages and Iframes as `Context`s?

`sandbox` documents and iframes have an
[opaque origin](https://html.spec.whatwg.org/multipage/browsers.html#concept-origin-opaque),
should we return them? What if they come from within the extension (or outside
in a Content Script)?

## Footnotes

<sup>1</sup>: Non-main threads in a
[Renderer](https://developer.chrome.com/blog/inside-browser-part3/) (where
service workers run in Chromium) cannot access DOM concepts directly (they are
only accessible from the main browser thread). Service workers run in these
threads therefore they do not have access to the JavaScript
[`Window`](http://go/mdn/API/Window#instance_properties) objects provided by
`extension.getViews()`. Supporting this access is likely years of software
engineer work to implement.
