---
layout: post
title: Introducing Giraffe.Htmx
date: 2021-11-09 16:06:00
author: Daniel
categories:
- [ Programming, .NET, F# ]
- [ Programming, htmx ]
- [ Projects, Giraffe.Htmx ]
tags:
- f#
- giraffe
- htmx
- view engine
---
[Giraffe][] is a library that sits atop ASP.NET Core and allows developers to build web applications in a functional style; `dotnet new giraffe` is _literally_ my starting point when I begin a new web application project. _(Rather than write three more sentences filled with effusive praise, I'll just leave it at that; it's great.)_ It also provides a view engine (that builds upon [Suave][]'s "experimental" view engine) which uses an F# <abbr title="Domain-Specific Language">DSL</abbr> to define HTML in a strongly-typed way. It has been incredibly efficient for a while, but with .NET's work over the past two releases at improving performance, and Giraffe's adoption of those techniques, it is lightning fast.

[htmx][] is a library that brings interactivity to HTML through the use of attributes and HTTP headers. Whereas projects like [Vue][], [Angular][], and [React][] prescribe completely different programming paradigms than traditional web development, htmx provides partial-page-swapping and progressive enhancement within straight HTML. This brings a lot of the benefits of the <abbr title="Single Page Application">SPA</abbr> architecture to vanilla HTML, without requiring a completely different paradigm than the one we have used on the web for 30 years. In practice, this greatly reduces the complexity required to produce an interactive web application.

The **Giraffe.Htmx** project provides a bridge between these two libraries. The project consists of two different NuGut packages.

- `Giraffe.Htmx` provides extensions to Giraffe (and its exposure of ASP.NET Core's `HttpContext`) that expose the [request headers][req-hdr] which htmx uses, and provides Giraffe-style `HttpHandler`s to set htmx's recognized [response headers][res-hdr]. The request headers are exposed as `Option`s, and if present, are converted to the expected type. Response headers can be set in a similar way (i.e., passing `true` instead of `"true"` for a boolean header).

```fsharp
  let myHandler : HttpHander =
    fun next ctx ->
      match ctx.Request.HxPrompt with
      | Some prompt -> ... // do something with the text the user provided
      | None -> ... // the user provided no text (likely was not even prompted)
      ...
```

- `Giraffe.ViewEngine.Htmx` extends Giraffe's view engine with [attribute][attrs] functions (ex. `_hxBoost` equates to `hx-boost="true"`) to generate HTML with htmx attributes. As with the headers, the values for each attribute are expected in their strongly-typed form, and the library handles the necessary string conversion. For attributes that have a defined set of values, there are also modules that provide those values; the example below demonstrates both the attributes and the `HxTrigger` module. 

```fsharp
  let autoLoad =
    div [ _hxGet "/this/endpoint"; _hxTrigger HxTrigger.Load ] [ str "Loading..." ]  
```

Head over to [the project site][site] for NuGet links and more examples!

p.s. As of this writing, the current (and only) version of this library is at v0.9.1. Both libraries should be ready for development use. For `Giraffe.ViewEngine.Htmx`, I intend to write helpers for `hx-headers` and `hx-vals` that will allow a list of `string * string` tuples to be passed. I also need to write READMEs for both NuGet packages. Once those are done, this will be v1-ready.


[Giraffe]: https://giraffe.wiki "Giraffe"
[Suave]: https://suave.io "Suave"
[htmx]: https://htmx.org "htmx"
[Vue]: https://vuejs.org "Vue.js"
[Angular]: https://angular.io "Angular"
[React]: https://reactjs.org "React"
[req-hdr]: https://htmx.org/docs/#request_headers "Request Headers | htmx Docs"
[res-hdr]: https://htmx.org/docs/#response_headers "Response Headers | htmx Docs"
[attrs]: https://htmx.org/docs/#attributes "Attributes | htmx Docs"
[site]: https://github.com/bit-badger/Giraffe.Htmx "Giraffe.Htmx | GitHub"
