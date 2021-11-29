---
layout: post
title: "A Tour of myPrayerJournal v3: The User Interface"
date: 2021-11-28 14:17:00
author: Daniel
categories:
- [ Programming, .NET, F# ]
- [ Programming, htmx ]
- [ Projects, myPrayerJournal ]
- [ Series, A Tour of myPrayerJournal v3 ]
tags:
- angular
- aurelia
- bootstrap
- elm
- ember
- f#
- giraffe
- html
- htmx
- javascript
- migration
- nuget
- post-redirect-get
- pug
- react
- single page application
- spa
- view engine
- vue
---

_NOTE: This is the second post in a series; see [the introduction][intro] for information on requirements and links to other posts in the series._

If you are a seasoned Single Page Application (SPA) framework developer, you likely think about interactivity in a particular way. Initially, I focused on replacing each interactive piece in isolation. In the end, though, requests for "pages" returned almost everything but the HTML head info and the displayed footer - and I was happy about it. Keep that in mind as I walk you down the path I have already traveled; keep an open mind, and read to the end before forming strong opinions either way.

## The $.05 Tour of Pug and Giraffe View Engine

Understanding the syntax of both [Pug][] and [Giraffe View Engine][gve] will help you if you click any of the source code example links. While a complete explanation of these two templating languages would make this long post much longer than it already is, here are some short examples of their syntax. Using a string variable `who` with the contents "World", we will show both languages rendering:

```html
<p id="example" class="greeting">Hello <strong>World</strong></p>
```

myPrayerJournal v2 used Pug templates in [Vue][] to render the user interface. Pug uses indentation-sensitive tag/content pairs (or blocks), with JavaScript syntax for attributes, to generate HTML. To generate the example paragraph, the shortest template would look like:

```pug
p.greeting(id="example") Hello #[strong= who]
```

myPrayerJournal v3 uses Giraffe View Engine, which uses F# lists to generate HTML from a very HTML-looking domain-specific language (DSL). The example paragraph would be generated with:

```fsharp
p [ _id "example"; _class "greeting" ] [ str "Hello "; strong [] [ who ] ]
```

Given those examples, let's dig into the conversion.

## The Menu

The menu across the top of the application was one of the first items I needed to convert. The menu needs to be different, depending on whether there is a user logged on or not. Also, if a user is logged on, the menu can still be different; the "Snoozed" menu item only appears if the user has any snoozed requests. The application uses [Auth0][] to manage users (which is how it is open to Microsoft and Google accounts), and I wanted to preserve this; my requests are tied to the ID provided by Auth0, so that did not need to change.

In the Vue version, the system used Auth0's SPA library that exposed whether there was a user logged on or not. Also, once a user was logged on, the API sent all the user's active requests, which included snoozed requests; once this API call returned, the application can turn on the "Snoozed" menu item. In the [htmx][] version, though, this information is all generated on the server. My initial process was to use an `hx-get` to get the menu HTML snippet, using an `hx-trigger` of `load` to fill in this spot of the page when the page was loaded. I also (initially) implemented a custom HTML header to include in responses, and if that header was found, I would trigger a refresh on the menu; the eventual solution included the navbar in "page" refreshes.

_(See the [Vue "Navigation" component][v2-nav] that became the [Giraffe View Engine "navbar" function][v3-nav])_

## "New Page" in htmx

This leads directly into a discussion of how myPrayerJournal is still considered a SPA. In the Vue version, "pages" were Vue Single-File Components (SFCs) under the `/components` directory. (In the years since myPrayerJournal v1, the default Vue template has changed to place these SFCs under `/views`, while `/components` is reserved for shared components.) These view components rendered into a custom component within the `main` tag (using Vue router's [`router-view` tag][v2-router]), while the `nav` component was reactive, based on the user logging on/off and snoozing requests.

In myPrayerJournal v3, "page" views target the [`#top` `section` element][v3-top]. If the request is for a full page load, the HTML `head` content is rendered, as is the `body`'s `footer` content; none of these change until a new version of the application is released. If the request is an htmx request, though, the only thing rendered is a new `#top` section, which includes the navigation bar and the page content. While this does approach a full "page load", there are some key differences:

- The page contents are refreshed based on one HTTP request (no extra request or processing required for the navbar);
- The HTML `head` content is responsible for most of the large HTTP requests, such as those for JavaScript libraries (and is excluded from non-full-page views);
- The page footer is not included.

Note the difference between the [full `view` layout][v3-full] and the [`partial` layout][v3-partial]. Also, within the application's request handlers, there is a [partial return function][v3-part-return] that determines whether this is an htmx-initiated page view request (in which case a partial view is returned) or a full page request (which returns the entire template).

## Updating the Page Title

One of the most unexpectedly-vexing parts of a SPA is determining how the browser's title bar will be updated when navigation occurs. _(I understand why it's challenging; what I do not understand is why it took major frameworks so long to devise a built-in way of handling this.)_ Coming from that world, I had originally implemented yet another custom header, pushing the title from the server, and used a request listener to update the title if the header was present. As I dug in further, though, I learned that htmx will update the document title any time a request payload has an HTML `title` element in its `head`. If you look at both layouts in the preceding paragraph, you'll notice that they include a `head` element with a `title` tag. This is how easy it should be, and with htmx, this is how easy it is.

At this point, there is a pattern emerging. The thought process behind an htmx-powered website is much different than a JavaScript-based SPA framework; and, in the majority of cases, it has been less complex. Now, let me contradict what I just said.

## POST-Redirect-GET

In myPrayerJournal v2, updating a prayer request followed this flow:

- Display the edit page, with the request details filled in
- When the user saved the request, return an empty `200 OK` response
- Using Vue, display a notification, refresh the journal, then re-render the page where the user had been when they clicked "Edit" (there are multiple places from which requests can be edited)

While there are no redirects here, this is the classic traditional-web-application scenario where the "POST-Redirect-GET" (P-R-G) pattern is used. By using this pattern, the "Back" button on the browser still works. If you try to go back to the result of a POST request, the browser will warn you that your action will result in the data being resubmitted (not usually what you want to do). By redirecting, though, the result of a POST becomes a GET, which does not change any data. For traditional web applications, this is the user-friendliest way to handle updates.

In the htmx examples, they show [an example of inline editing][in-edit]. This led to my first plan - change the request edit "page" to be a component, where the HTML for the displayed list was replaced by the form, and then the "Save" action returns the new HTML. This requires no P-R-G, as these actions have no effect on the "Back" button. It worked fine, but there were some things that weren't quite right:

- New requests needed their own page; I was going to have to duplicate the edit form for the "new" page, and introduce some complexity in determining how to render the results.
- Some updates required refreshing the list of requests, not just replacing the text and action buttons.

At this point, I was also starting to realize "if you think something is hard to do in htmx, you probably aren't trying to do it correctly." So, I decided to try to replicate the "edit page" flow of v2 in v3. Creating [the page][v3-edit] was easy enough, and I was able to use the `returnTo` parameter in the function to both provide a "Cancel" button and redirect the user to the right place after saving. Easy, right? Well... Not quite. 

htmx uses `XMLHttpRequest` (XHR) to send its requests, which has some interesting behavior; it follows redirects! When I submitted my form, it received the request (with htmx's `HX-Request` header set), and the server returned the redirect. XHR saw this, and followed it; however, it used the same method. (It was POSTing to the new URL.) The fix for this, though, was not easy to find, but easy to implement; use HTTP response code `303` (see other) instead of `307` (moved temporarily). Using this, combined with using `hx-target="#top"` on the form, allowed the P-R-G pattern to work successfully without double-POSTing and without a full-page refresh. 

## htmx Support in Giraffe and Giraffe View Engine

As I developed this, I was also building up extensions for Giraffe to handle the htmx request and response headers, as well as the attributes needed to generate htmx-aware markup from Giraffe View Engine. [This project][g-h], called Giraffe.Htmx, is now published on NuGet. There are two packages; `Giraffe.Htmx` provides server-side support for the request and response headers, and `Giraffe.ViewEngine.Htmx` provides support for generating htmx attributes. I [wrote about it][g-h-rel] when it was released, so I won't rehash the entire thing here.

## Final UI Thoughts

htmx is much less complex than any other front-end JavaScript SPA framework I have ever used - which, for context, includes [Angular][], Vue, [React][], [Ember][], [Aurelia][], and [Elm][]. Both in development and in production use, I cannot tell that the payloads are slightly larger; navigation is fast and smooth. Though I have yet to change anything since going live with myPrayerJournal v3, I know that maintenance will be quite straightforward (to be further explored in the conclusion post).

The UI for myPrayerJournal uses [Bootstrap][], a UI framework which has its own script, and htmx plays quite nicely with it. The [next post][part2] in this series will describe how I interact with both Bootstrap and htmx, using modals and toasts on this "traditional" web application.


[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[Pug]: https://pugjs.org/ "Pug"
[gve]: https://giraffe.wiki/view-engine "Giraffe View Engine"
[Vue]: https://vuejs.org "Vue.js"
[Auth0]: https://auth0.com "Auth0"
[htmx]: https://htmx.org "htmx"
[v2-nav]: https://github.com/bit-badger/myPrayerJournal/blob/2.2/src/app/src/components/common/Navigation.vue "myPrayerJournal v2 Navigation Vue component"
[v3-nav]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Views/Layout.fs#L39 "myPrayerJournal v3 navbar function"
[v2-router]: https://github.com/bit-badger/myPrayerJournal/blob/2.2/src/app/src/App.vue#L16 "myPrayerJournal v2 router-view tag"
[v3-top]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Views/Layout.fs#L140 "myPrayerJournal v3 #top section"
[v3-full]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Views/Layout.fs#L136 "myPrayerJournal v3 full layout"
[v3-partial]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Views/Layout.fs#L147 "myPrayerJournal v3 partial layout"
[v3-part-return]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Handlers.fs#L162 "myPrayerJournal v3 partial return function"
[in-edit]: https://htmx.org/examples/click-to-edit/ "Click to Edit | htmx"
[v3-edit]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Views/Request.fs#L139 "myPrayerJournal v3 Request Edit page"
[g-h]: https://github.com/bit-badger/Giraffe.Htmx "Giraffe.Htmx"
[g-h-rel]: /2021/introducing-giraffe-htmx.html "Introducing Giraffe.Htmx | The Bit Badger Blog"
[Angular]: https://angular.io "Angular"
[React]: https://reactjs.org "React"
[Ember]: https://emberjs.com "Ember"
[Aurelia]: https://aurelia.io "Aurelia"
[Elm]: https://elm-lang.org "Elm"
[Bootstrap]: https://getbootstrap.com "Bootstrap"
[part2]: /2021/a-tour-of-myprayerjournal-v3/bootstrap-integration.html "A Tour of myPrayerJournal v3: Bootstrap Integration | The Bit Badger Blog"
