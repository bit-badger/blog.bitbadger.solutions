---
layout: post
title: "A Tour of myPrayerJournal v3: Conclusion"
date: 2021-12-04 14:17:00
author: Daniel
categories:
- [ Databases, LiteDB ]
- [ Programming, .NET, F# ]
- [ Programming, htmx ]
- [ Projects, Giraffe.Htmx ]
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

_NOTE: This is the final post in a series; see [the introduction][intro] for information on requirements and links to other posts in the series._

We've gone in depth on several different aspects of this application and the technologies it uses. Now, let's zoom out and look at some big-picture lessons learned.

## What I Liked

### Simplification Via htmx

One of the key concepts in a Representational State Transfer (REST) <abbr title="Application Programming Interface">API</abbr> is that of Hypermedia As the Engine of Application State (HATEOAS). In short, this means that the state of an application is held within the hypermedia that is exchanged between client and server; and, in practice, the server is responsible for altering that state. This is slightly different from the concept of sessions, and completely different from the <abbr title="JavaScript Object Notation">JSON</abbr> API / JavaScript framework model.

_(This is a near over-simplification; the paper that initially proposed these concepts earned its author a doctoral degree.)_

The simplicity of this model is great; and, when I say "simplicity," I am speaking of a lack of complexity, not a naivete of approach. The amount of complexity and synchronization I was able to remove between myPrayerJournal v2 and v3 was quite large. State management used to be the most complex part of the application; now, it's the rendering of HTML. Since that HTML is what controls the state, this makes sense. However, I have 25 years of experience writing HTML, and the language itself is not terribly complex, even at its most complex.

### LiteDB

This was a very simple application - and, despite its being open for any user with a Google or Microsoft account, I have been the only regular user of the application. LiteDB's setup was easy, implementation was easy, and it performs really well. I suspect this would be the case with many concurrent users as well, which speaks well to its abilities. If the application were to grow, I could even theoretically create a database file per user, and back up the directory instead of a specific file. As with [htmx][], the lack of complexity makes the application easily maintainable.

## What I Learned

### htmx Support in .NET

I mentioned [the project I developed as a part of this effort][g-h], and that I had become aware of htmx on an episode of _.NET Rocks!_. `Giraffe.Htmx` is very F#-centric, using features of that language that are not exposed in C# or VB.NET. However, there are two packages that work with the standard ASP.NET Core method of development. [`Htmx`][a-h] provides server-side support for the htmx request and response headers, similar to `Giraffe.Htmx`, and [`Htmx.TagHelpers`][a-h-h] contains tag helpers for use in Razor, similar to `Giraffe.ViewEngine.Htmx`. Both are written by [Khalid Abuhakmeh][ka], a developer advocate at [JetBrains][] (which generously [licensed their tools to this project][jb-oss], and developed [the best developer font ever][jb-mono]).

While I did not use these projects, I did look at the source, and they look good. Besides, the best way to improve an open-source library is to use it, and provide good feedback.

### Write about Your Code

Yes, [I'm cheating a bit with this one][v1-write], but it's still true. Writing about your code has several benefits:

- You understand your code more fully.
- Others can see not just the code you wrote, but understand the thought process behind it.
- Readers can provide you feedback. _(This may not always seem helpful; regardless of its tone, though, thinking through whether the point of their critique is justified can help you learn.)_

And, really, knowledge sharing is what makes the open-source ecosystem work. Closed / proprietary projects have their place, but if you do something interesting, write about it!

## What Could Be Better

### Deferred Features

There were 2 changes I had originally planned for myPrayerJournal v3 that did not get accomplished. One is a new "pray through the requests" view, with a distraction-free next-card-up presentation. The other is that updating requests sends them to the bottom of the list, even if they have not been marked as prayed; this will require calculating a separate "last prayed" date instead of using the "as of" date from the latest history entry.

This migration introduced a third deferred change. When v1/v2 ran in the browser, the dates and times were displayed in the user's local timezone. With the HTML being generated on the server, though, dates and times are now displayed in UTC. The purpose of the application is to focus the user's attention on their prayer request, not make them have to do timezone math in their head! htmx has an `hx-headers` attribute that specifies headers to pass along with the request; I plan to use [a JavaScript call][js-intl] to set a header on the `body` tag when a full page loads (`hx-headers` is inherited), then use that timezone to adjust it back to the user's current timezone.

### That LiteDB Mapping

I did a good bit of tap-dancing in the [LiteDB data model and mapping descriptions][part3], mildly defending the design decisions I'd made there. The recurrence should be designed differently, and there should be individual type mappings rather than mapping the entire document. Yes, it worked for my purpose, and this project was more about Vue to htmx than ensuring a complete F#-to-LiteDB mapping of domain types. As I think about implementing the features about, though, I believe I will end up fixing those issues as well. _(This is another side effect of writing about your code; sometimes, you wish you had written something differently.)_

===

Thank you for joining me on this tour; I hope it has been enjoyable, and maybe even educational.


[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[htmx]: https://htmx.org "htmx"
[g-h]: https://github.com/bit-badger/Giraffe.Htmx "Giraffe.Htmx"
[a-h]: https://www.nuget.org/packages/Htmx/ "Htmx | NuGet"
[a-h-h]: https://www.nuget.org/packages/Htmx.TagHelpers/ "Htmx.TagHelpers | NuGet"
[ka]: https://khalidabuhakmeh.com "Khalid Abuhakmeh"
[JetBrains]: https://www.jetbrains.com "JetBrains"
[jb-oss]: https://www.jetbrains.com/community/opensource/#support "Licenses for Open Source Development | JetBrains"
[jb-mono]: https://www.jetbrains.com/lp/mono/ "JetBrains Mono"
[v1-write]: /2018/a-tour-of-myprayerjournal/conclusion.html#Write-about-Your-Code "A Tour of myPrayerJournal: Conclusion | The Bit Badger Blog"
[js-intl]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat/resolvedOptions "Intl.DateTimeFormat.prototype.resolvedOptions | MDN"
[part3]: /2021/a-tour-of-myprayerjournal-v3/the-data-store.html "A Tour of myPrayerJournal v3: The Data Store | The Bit Badger Blog"
