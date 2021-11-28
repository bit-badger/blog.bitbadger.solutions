---
layout: post
title: "A Tour of myPrayerJournal v3: Introduction"
date: 2021-11-26 16:04:00
author: Daniel
categories:
- [ Databases, LiteDB ]
- [ Programming, .NET, F# ]
- [ Programming, htmx ]
- [ Projects, myPrayerJournal ]
- [ Series, A Tour of myPrayerJournal v3 ]
tags:
- f#
- giraffe
- htmx
- introduction
- javascript
- journaling
- litedb
- prayer
- requirements
- vue
---
This is the first of 5 posts in this series.

- **Part 0: Introduction** _(this post)_
- **[Part 1: The User Interface][part1]** - A look at htmx and Giraffe working together to create the web UI
- **Part 2: Using Bootstrap** - A little bit of JavaScript goes a long way
- **Part 3: The Data Store** - Migration to and usage of LiteDB
- **Part 4: Conclusion** - Lessons learned and areas for improvement

## Background

Around 3 years ago, I wrote an 8-part series called ["A Tour of myPrayerJournal"][tour1], recounting the decisions and implementation of its initial release. Version 2 did not get its own tour, as it used a similar architecture. There were also some nagging library issues that were never resolved, leading to v2 being an overall unsatisfying step in the evolution of this application.

When [Vue][] v3 was announced, this sounded like a great opportunity, with first-class TypeScript support and a new component syntax that promised better performance and a better developer experience. This past summer, I completed a project with the mature Vue v3 framework, and was generally pleased with the results. Just after I returned to my previously abandoned migration attempt on this project (with early Vue v3 support), I heard about **htmx**. With a few attributes, and a server that can handle a few HTTP headers, you can build an interactive site, with performance rivaling or exceeding that of the typical Single Page Application (SPA) - or, at this point, so they claimed.

I also picked up **LiteDB** on another project over the summer, and it worked well. I thought, why not give these technologies a try, and see if I would like result?

_(SPOILER ALERT: I did!)_

## The Requirements

Requirements for v3 were, for the most part, to update the application to Vue v3. Without rehashing the entire list (see the other intro post), the basic idea is that a prayer request is represented by a card, and this card keeps up with all changes made to it. Also, the system can present the cards that are active, arranged with the oldest action date first, and allow you to tick through the cards. (This is the flow to enable the user to "pray through their list.")

The goal is to remain a minimalist program; the focus should be on prayer, not using a website. To that end, I had envisioned a "one-at-a-time" scenario that would clear out distractions and present the cards in the same order. I had also planned to separate the "last prayed" date from the "last activity" date; currently, updating the text of a request moves it to the bottom of the stack. However, both of these improvements were deferred to v3.1; v3 restores the (adequate) functionality of v1, while being much lighter-weight.

## The Tech Stack

This stack did not go through nearly as many iterations as v1.

**[Giraffe][]** is a library that enables F# developers to create ASP.NET Core endpoints in a functional style. It's a mature library (v1 used Giraffe!), and continues to be improved. It also provides an optional "Giraffe View Engine," which will get more attention in the user interface post; the views for v3 are produced via this view engine.

**[htmx][]** is a JavaScript library that asks... well, several questions. Why should links and buttons be the only interactive elements? Why should you have to replace the whole page every time? What would HTML look like if it had been developed the way a typical programming language would be? It uses a small set of attributes to answer these questions differently, making interactive sites possible without writing any JavaScript. (The custom JavaScript file in v3 is 82 lines, including comments - and the majority of that is [Bootstrap][] interaction.)

Since, in the htmx way, the web server returns rendered HTML, the requests can be a bit larger than the equivalent API calls that return JSON for a SPA framework to render. However, this is offset somewhat by the fact that the browser just has to swap that HTML fragment in; the processing is faster and much less complex.

What really swung me over the fence to giving it a shot, though, was a point Carson (the author of the library) made while talking with Carl and Richard on the _[.NET Rocks!][]_ podcast. Having a server render the HTML, and the browser merely displaying it, keeps your application logic on the server; the only JavaScript you need to write is what is required for the user interface. This eliminates a host of synchronization issues with SPAs and their associated APIs - duplicating shapes of data, ensuring calculations are in sync, etc. It also keeps your application logic from needing to be exposed to the public Internet; this doesn't entirely prevent exploits, but the prospective hacker doesn't start with a full copy of your code.

**[LiteDB][]** could be described as SQLite for documents. Collections of Plain-Old CLR Objects (POCOs) can be stored, retrieved, searched, indexed, and deleted, all while running in the current process, and requiring no separate database server install. While it does not require any special configuration to do this, it does also provide the ability to transform these objects. This gives complete control as to how much or how little transformation you want to specify; and, as we'll see in part 3, this came in handy for this application.

## Where We Go from Here

In the next post, we'll take a look at Giraffe, its View Engine, htmx, and how they all work together. The post after that will dive into the aforementioned 82 lines of JavaScript to see how we can control Bootstrap's client/browser behavior from the server. After that, we'll dig in on LiteDB, to include how we serialize some common F# constructs. Finally, we'll wrap up the series with overarching lessons learned, and other thoughts which may not fit nicely into one of the other posts.


[part1]: /2021/a-tour-of-myprayerjournal-v3/the-user-interface.html "A Tour of myPrayerJournal v3: The User Interface | The Bit Badger Blog"
[tour1]: /2018/a-tour-of-myprayerjournal/introduction.html "A Tour of myPrayerJournal: Introduction | The Bit Badger Blog"
[Vue]: https://vuejs.org "Vue"
[Giraffe]: https://giraffe.wiki "Giraffe"
[htmx]: https://htmx.org "htmx"
[Bootstrap]: https://getbootstrap.com "Bootstrap"
[LiteDB]: https://litedb.org "LiteDB"
[.NET Rocks!]: https://www.dotnetrocks.com/?show=1749 "htmx with Carson Gross | .NET Rocks!"
