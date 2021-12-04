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

## Simplification Via htmx

One of the key concepts in a Representational State Transfer (REST) <abbr title="Application Programming Interface">API</abbr> is that of Hypermedia As the Engine of Application State (HATEOAS). In short, this means that the state of an application is held within the hypermedia that is exchanged between client and server; and, in practice, the server is responsible for altering that state. This is slightly different from the concept of sessions, and completely different from the <abbr title="JavaScript Object Notation">JSON</abbr> API / JavaScript framework model.

_(This is a near over-simplification; the paper that initially proposed these concepts earned its author a doctoral degree.)_

The simplicity of this model is great; and, when I say "simplicity," I am speaking of a lack of complexity, not a naivete of approach. The amount of complexity and synchronization I was able to remove between myPrayerJournal v2 and v3 was quite large. State management used to be the most complex part of the application; now, it's the rendering of HTML. Since that HTML is what controls the state, this makes sense. However, I have 25 years of experience writing HTML, and the language itself is not terribly complex, even at its most complex.

## htmx Support in .NET

I mentioned [the project I developed as a part of this effort][g-h], and that I had become aware of [htmx][] on an episode of _.NET Rocks!_. `Giraffe.Htmx` is very F#-centric, using features of that language that are not exposed in C# or VB.NET. However, there are two packages that work with the standard ASP.NET Core method of development. [`Htmx`][a-h] provides server-side support for the htmx request and response headers, similar to `Giraffe.Htmx`, and [`Htmx.TagHelpers`][a-h-h] contains tag helpers for use in Razor, similar to `Giraffe.ViewEngine.Htmx`. Both are written by [Khalid Abuhakmeh][ka], a developer advocate at [JetBrains][] (which generously [licensed their tools to this project][jb-oss], and developed [the best developer font ever][jb-mono]).

While I did not use these projects, I did look at the source, and they look good. Besides, the best way to improve an open-source library is to use it, and provide good feedback.



[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[gve]: https://giraffe.wiki/view-engine "Giraffe View Engine"
[g-h]: https://github.com/bit-badger/Giraffe.Htmx "Giraffe.Htmx"
[htmx]: https://htmx.org "htmx"
[a-h]: https://www.nuget.org/packages/Htmx/ "Htmx | NuGet"
[a-h-h]: https://www.nuget.org/packages/Htmx.TagHelpers/ "Htmx.TagHelpers | NuGet"
[ka]: https://khalidabuhakmeh.com "Khalid Abuhakmeh"
[JetBrains]: https://www.jetbrains.com "JetBrains"
[jb-oss]: https://www.jetbrains.com/community/opensource/#support "Licenses for Open Source Development | JetBrains"
[jb-mono]: https://www.jetbrains.com/lp/mono/ "JetBrains Mono"
[g-h-rel]: /2021/introducing-giraffe-htmx.html "Introducing Giraffe.Htmx | The Bit Badger Blog"
[Angular]: https://angular.io "Angular"
[React]: https://reactjs.org "React"
[Ember]: https://emberjs.com "Ember"
[Aurelia]: https://aurelia.io "Aurelia"
[Elm]: https://elm-lang.org "Elm"
[Bootstrap]: https://getbootstrap.com "Bootstrap"
[part2]: /2021/a-tour-of-myprayerjournal-v3/bootstrap-integration.html "A Tour of myPrayerJournal v3: Bootstrap Integration | The Bit Badger Blog"
[a-h]: https://www.nuget.org/packages/Htmx/ "Htmx | NuGet"
