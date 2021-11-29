---
layout: post
title: "A Tour of myPrayerJournal v3: Bootstrap Integration"
date: 2021-11-29 14:17:00
author: Daniel
categories:
- [ Programming, .NET, F# ]
- [ Programming, htmx ]
- [ Projects, myPrayerJournal ]
- [ Series, A Tour of myPrayerJournal v3 ]
tags:
- bootstrap
- f#
- giraffe
- html
- htmx
- javascript
- migration
- single page application
- spa
- view engine
- vue
---

_NOTE: This is the third post in a series; see [the introduction][intro] for information on requirements and links to other posts in the series._

Many Single Page Application (SPA) frameworks not only allow you to never do a full refresh of your page for the duration of a visit, many also include (or have plugins for) <abbr title="Cascading Style Sheets">CSS</abbr> transitions and effects. From the user's perspective, this is one of their best features. One might not think that a framework like [htmx][], which simply swaps out sections of the page, would have this; but one would be wrong. I did not dig into those aspects of htmx while I was migrating myPrayerJournal from v2 to v3; however, I will highlight the htmx way to do this in [last section of this post][css-in-htmx].

myPrayerJournal v2 used a [Vue][] plugin that provided Bootstrap v4 support; myPrayerJournal v3 uses [Bootstrap][] v5. The main motivation I had to remain with Bootstrap was that I liked the actual appearance, and I know how it works; the majority of the "learning" on this project was in the htmx realm. But, before I dig in to the implementation, let me briefly explain the framework.   

## About Bootstrap

Bootstrap was originally called Twitter Bootstrap; it was the CSS framework that Twitter developed in their early iterations. It was, by far, the most popular framework at the time, and it was innovative in its grid layout system. Long before there was browser support for the styles that make layouts much easier to develop, and more responsive to differing screen sizes, Bootstrap's grid layout and size breakpoints made it easy to build a website that worked both on the desktop or on a phone. Of course, there is a limit to what you can do with styling, so Bootstrap also has a JavaScript library that augments these styles, enabling the interactivity to which the modern web user is accustomed.

Version 5 of Bootstrap continues this tradition; however, it brings in even more utility classes, and supports Flex layouts as well. It is a mature library that continues to be maintained, and the project's philosophy seems to be a "just enough" - it's not going to do everything for everyone, but in the majority of cases, it has exactly what the developer needs. It is not a bloated library that needs tree-shaking to avoid a ridiculous download size.

It is, by far, the largest payload in the initial page request:

- Bootstrap - 48.6 kB (CSS is 24.8 kB; JavaScript is 23.8 kB, deferred until after render)
- htmx - 11.8 kB
- myPrayerJournal - 4.4 kB (CSS is 1.2 kB, JavaScript is 3.2 kB)

However, this gets the entire style and script, and allows us to use their layouts and interactive components. But, how do we get that interactivity from the server?

## Hooking in to the htmx Request Pipeline 

htmx provides [several events][htmx-evts] to which an application can listen. In myPrayerJournal v3, [I used `htmx:afterOnLoad`][v3-onload] because I did not need the new content to be swapped in yet when the function fired. There are `afterSwap` and `afterSettle` events which will fire once those events have occurred, if you need to defer processing until those are complete.

There are two different Bootstrap script-driven components myPrayerJournal uses; let's take a look at toasts.

## A Toast to htmx

[Toasts][bs-toast] are pop-up notifications that appear on the screen, usually for a short time, then fade out. In some cases, particularly if the toast is alerting the user to an error, it will stay on the screen until the user dismisses it, usually by clicking an "x" in the upper right-hand corner (even if the developer used a Mac!). Bootstrap provides a host of options for their toast component; for our uses, though, we will:

- Place toasts in the bottom right-hand corner;
- Allow multiple toasts to be visible at once;
- Auto-hide success toasts, require others to be dismissed manually.

There are several different aspects that make this work.

### The Toaster

Just like toast comes out of a toaster, our toasts need a place from which to emerge. In the [prior post][prior], I mentioned that the footer does not get reloaded when a "page" request is made. There is also an element above the footer that also remains across these requests - defined here as the "toaster" (my term, not Bootstrap's).

```fsharp
/// Element used to display toasts
let toaster =
  div [ _ariaLive "polite"; _ariaAtomic "true"; _id "toastHost" ] [
    div [ _class "toast-container position-absolute p-3 bottom-0 end-0"; _id "toasts" ] []
    ]
```

This renders two empty `div`s with the appropriate style attributes; toasts placed here will display as we want them to.

### Creating a Toast

Bootstrap provides `data-` attributes that can make toasts appear; however, since we are creating these in script, we need to use their JavaScript functions. The message coming from the server has the format `TYPE|||The message`. Let's look at [the showToast function][v3-toast] (the largest custom JavaScript function in the entire application):

```javascript
showToast (message) {
  const [level, msg] = message.split("|||")
  
  let header
  if (level !== "success") {
    const heading = typ => `<span class="me-auto"><strong>${typ.toUpperCase()}</strong></span>`
    
    header = document.createElement("div")
    header.className = "toast-header"
    header.innerHTML = heading(level === "warning" ? level : "error")
    
    const close = document.createElement("button")
    close.type = "button"
    close.className = "btn-close"
    close.setAttribute("data-bs-dismiss", "toast")
    close.setAttribute("aria-label", "Close")
    header.appendChild(close)
  }

  const body = document.createElement("div")
  body.className = "toast-body"
  body.innerText = msg
  
  const toastEl = document.createElement("div")
  toastEl.className = `toast bg-${level === "error" ? "danger" : level} text-white`
  toastEl.setAttribute("role", "alert")
  toastEl.setAttribute("aria-live", "assertlive")
  toastEl.setAttribute("aria-atomic", "true")
  toastEl.addEventListener("hidden.bs.toast", e => e.target.remove())
  if (header) toastEl.appendChild(header)
  
  toastEl.appendChild(body)
  document.getElementById("toasts").appendChild(toastEl)
  new bootstrap.Toast(toastEl, { autohide: level === "success" }).show()
}
```

Here's what's going on in the code above:

- Line 2 splits the level from the message
- Lines 4-18 (`let header`) create a header and close button if the message is not a success
- Lines 20-22 (`const body`) create the body `div` with attributes Bootstrap's styling expects
- Lines 24-28 (`const toastEl`) create the `div` that will contain the toast
- Line 29 adds an event handler to remove the element from the DOM once the toast is hidden
- Lines 30 and 32 add the optional header and mandatory body to the toast `div`
- Line 33 adds the toast to the page (within the `toasts` inner `div` defined above)
- Line 34 initializes the Bootstrap JavaScript component, auto-hiding on success, and shows the toast

_(If you've never used JavaScript to create elements that are added to an HTML document, this probably looks weird and verbose; if you have, you look at it and say "they're not wrong...")_

So, we have our toaster, we know how to put ~~bread~~ notifications in it - but how do we get the notifications from the server?

### Receiving the Toast

The code to handle this is part of the `htmx:afterOnLoad` handler:

```javascript
htmx.on("htmx:afterOnLoad", function (evt) {
  const hdrs = evt.detail.xhr.getAllResponseHeaders()
  // Show a message if there was one in the response
  if (hdrs.indexOf("x-toast") >= 0) {
    mpj.showToast(evt.detail.xhr.getResponseHeader("x-toast"))
  }
  // ...
})
```

This looks for a custom HTTP header of `X-Toast` (all headers are lowercase from that `xhr` call), and if it's found, we pass the value of that header to the function above. This check occurs after every htmx network request, so there is nothing special to configure; "page" requests are not the only requests capable of returning a toast notification.

There is one more part; how does the toast get to the browser?

### Server-Sent Toast

The last paragraph gave it away; we set a header on the response. This seems straightforward, but [once again, POST-Redirect-GET][part1-prg] (P-R-G) complicates things. Here are the final two lines of the successful path of [the request update handler][v3-upd8]:

```fsharp
Messages.pushSuccess ctx "Prayer request updated successfully" nextUrl
return! seeOther nextUrl next ctx
```

If we set a message in the response header, then redirect (remember that `XMLHttpRequest` handles redirects silently), the header gets lost in the redirect. Here, `Messages.pushSuccess` places the success message in a dictionary, indexed by the user's ID. Within the function that renders every result (partial, "page"-like, or full results), this dictionary is checked for a message and URL, and if one exists, it includes it. (If it is returned to the function below, it has already been removed from the dictionary.)

```fsharp
/// Send a partial result if this is not a full page load (does not append no-cache headers)
let partialStatic (pageTitle : string) content : HttpHandler =
  fun next ctx -> backgroundTask {
    let  isPartial = ctx.Request.IsHtmx && not ctx.Request.IsHtmxRefresh
    let! pageCtx   = pageContext ctx pageTitle content
    let  view      = (match isPartial with true -> partial | false -> view) pageCtx
    return! 
      (next, ctx)
      ||> match user ctx with
          | Some u ->
              match Messages.pop u with
              | Some (msg, url) -> setHttpHeader "X-Toast" msg >=> withHxPush url >=> writeView view
              | None -> writeView view
          | None -> writeView view
    }
```

A quick overview of this function:

- Line 4 determines if this an htmx boosted request (a "page"-like requests)
- Line 5 creates a rendering context for the page
- Line 6 renders the view to a string, calling `partial` or `view` with the page rendering context
- Lines 10-13 are only executed if a user is logged on, and line 12 is the one that appends a message and a new URL

> A quick note about line 12: the `>=>` operator joins Giraffe `HttpHandler`s together. An `HttpHandler` takes an `HttpContext` and the next function to be executed, and returns a `Task<HttpContext option>` (an asynchronous call that may or may not return a context). If there is no context returned, the chain stops; however, the function can return an altered context. It is good practice for an `HttpHandler` to make a single change to the context; this keeps them simple, and allows them to be plugged in however the developer desires. Thus, the `setHttpHeader` call adds the `X-Toast` header, the `withHxPush` call adds the `HX-Push` header, and the `writeView` call sets the response body to the rendered view.

The new URL part does not actually make the browser do anything; it simply pushes the given URL onto the browser's history stack. Technically, the browser receives the content from the P-R-G as the response to its POST; as we're replacing the current page, though, we need to make sure the URL stays in sync.

Of note is that not all toasts are this complex. For example, the "cancel snooze" handler return looks like this:

```fsharp
return! (withSuccessMessage "Request unsnoozed" >=> Components.requestItem requestId) next ctx
```

...while the `withSuccessMessage` handler is:

```fsharp
/// Add a success message header to the response
let withSuccessMessage : string -> HttpHandler =
  sprintf "success|||%s" >> setHttpHeader "X-Toast"
```

No dictionary, no redirect, just a single response that will show a toast.

You made it - the toast section is toast! There is one more interesting interaction, though; that of the modal dialog.

## Modal Dialogs

Bootstrap's [implementation of modal dialogs][bs-modal] also uses JavaScript; however, for the purposes of the modals used in myPrayerJournal v3, we can use the `data-` attributes to show them. Here is the view for a modal dialog that allows the user to snooze a request:

```fsharp
div [
  _id             "snoozeModal"
  _class          "modal fade"
  _tabindex       "-1"
  _ariaLabelledBy "snoozeModalLabel"
  _ariaHidden     "true"
  ] [
  div [ _class "modal-dialog modal-sm" ] [
    div [ _class "modal-content" ] [
      div [ _class "modal-header" ] [
        h5 [ _class "modal-title"; _id "snoozeModalLabel" ] [ str "Snooze Prayer Request" ]
        button [ _type "button"; _class "btn-close"; _data "bs-dismiss" "modal"; _ariaLabel "Close" ] []
        ]
      div [ _class "modal-body"; _id "snoozeBody" ] [ ]
      div [ _class "modal-footer" ] [
        button [ _type "button"; _id "snoozeDismiss"; _class "btn btn-secondary"; _data "bs-dismiss" "modal" ] [
          str "Close"
          ]
        ]
      ]
    ]
  ]
```

Notice that `#snoozeBody` is empty; we fill that when the user clicks the snooze icon:

```fsharp
button [
  _type     "button"
  _class    "btn btn-secondary"
  _title    "Snooze Request"
  _data     "bs-toggle" "modal"
  _data     "bs-target" "#snoozeModal"
  _hxGet    $"/components/request/{reqId}/snooze"
  _hxTarget "#snoozeBody"
  _hxSwap   HxSwap.InnerHtml
  ] [ icon "schedule" ]
```

This uses `data-bs-toggle` and `data-bs-target`, Bootstrap attributes, to show the modal. It also uses `hx-get` to load the snooze form for that particular request, with `hx-target` targeting the `#snoozeBody` `div` from the modal definition. Here is how that form is defined:

```fsharp
/// The snooze edit form
let snooze requestId =
  let today = System.DateTime.Today.ToString "yyyy-MM-dd"
  form [
    _hxPatch  $"/request/{RequestId.toString requestId}/snooze"
    _hxTarget "#journalItems"
    _hxSwap   HxSwap.OuterHtml
    ] [
    div [ _class "form-floating pb-3" ] [
      input [ _type "date"; _id "until"; _name "until"; _class "form-control"; _min today; _required ]
      label [ _for "until" ] [ str "Until" ]
      ]
    p [ _class "text-end mb-0" ] [ button [ _type "submit"; _class "btn btn-primary" ] [ str "Snooze" ] ]
    ]
```

Here, the form uses `hx-patch` to submit the data to the snooze endpoint. The target for the response, though, is `#journalItems`; this is the element that holds all of the prayer request cards. Snoozing a request will remove it from the active list, so the list needs to be refreshed; this will make that happen.

Look back at the modal definition; at the bottom, there is a "Close" button. We will use this to dismiss the modal once the update succeeds. In the Giraffe handler to snooze a request, here is its `return` statement:

```fsharp
return!
  (withSuccessMessage $"Request snoozed until {until.until}"
  >=> hideModal "snooze"
  >=> Components.journalItems) next ctx
```

Notice that `hideModal` handler?

```fsharp
/// Hide a modal window when the response is sent
let hideModal (name : string) : HttpHandler =
  setHttpHeader "X-Hide-Modal" name
```

Yes, it's another HTTP header! One can certainly get carried away with custom HTTP headers, but their very existence is to communicate with the client (browser) outside of the visible content of the page. Here, we're passing the name "snooze" to this header; in our `htmx:afterOnLoad` handler, we'll consume this header:

```javascript
htmx.on("htmx:afterOnLoad", function (evt) {
  const hdrs = evt.detail.xhr.getAllResponseHeaders()
  // ...
  // Hide a modal window if requested
  if (hdrs.indexOf("x-hide-modal") >= 0) {
    document.getElementById(evt.detail.xhr.getResponseHeader("x-hide-modal") + "Dismiss").click()
  }
})
```

The "Close" button on our modal was given the `id` of `snoozeDismiss`; this mimics the user clicking the button, which Bootstrap's `data-` attributes handle from there. Of all the design choices and implementations I did in this conversion, this part strikes me as the most "hack"y. However, I did try to hook into the Bootstrap modal itself, and hide it via script; however, it didn't like initializing a modal a second time, and I could not get a reference to it from the `htmx:afterOnLoad` handler. Clicking the button works, though, even when it's done from script.

## CSS Transitions in htmx

This post has already gotten much longer than I had planned, but I wanted to make sure I covered this briefly.

- When htmx requests are in flight, the framework makes it easy to [show indicators][ind].
- I mentioned swapping and settling when discussing the events htmx exposes. The way this is done, [CSS transitions][trans] will render as expected. They have [a host of examples][exmpls] to spark your imagination.

As I was looking for a UI conversion, I did not end up using these; however, this shows that htmx is a true batteries-included framework.

---

Up next, we'll step away from the front end and dig into LiteDB.


[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[htmx]: https://htmx.org "htmx"
[css-in-htmx]: #the-last-section "CSS Interactivity with htmx | A Tour of myPrayerJournal v3: Bootstrap Integration | The Bit Badger Blog"
[Vue]: https://vuejs.org "Vue.js"
[Bootstrap]: https://getbootstrap.com "Bootstrap"
[htmx-evts]: https://htmx.org/reference/#events "Events | Reference | htmx"
[v3-onload]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/wwwroot/script/mpj.js#L72 "myPrayerJournal v3 htmx:afterOnLoad function"
[bs-toast]: https://getbootstrap.com/docs/5.1/components/toasts/ "Toasts | Bootstrap"
[prior]: /2021/a-tour-of-myprayerjournal-v3/the-user-interface.html#%E2%80%9CNew-Page%E2%80%9D-in-htmx "A Tour of myPrayerJournal v3: The User Interface | The Bit Badger Blog"
[v3-toast]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/wwwroot/script/mpj.js#L9 "myPrayerJournal v3 showToast function"
[part1-prg]: /2021/a-tour-of-myprayerjournal-v3/the-user-interface.html#POST-Redirect-GET "A Tour of myPrayerJournal v3: The User Interface | The Bit Badger Blog"
[v3-upd8]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Handlers.fs#L503 "myPrayerJournal v3 request update handler"
[bs-modal]: https://getbootstrap.com/docs/5.1/components/modal/ "Modal | Bootstrap"
[ind]: https://htmx.org/docs/#indicators "Request Indicators | Docs | htmx"
[trans]: https://htmx.org/docs/#css_transitions "CSS Transitions | Docs | htmx"
[exmpls]: https://htmx.org/examples/animations/ "Animations | Examples | htmx"
