---
layout: post
title: "A Tour of myPrayerJournal v3: The Data Store"
date: 2021-11-30 14:17:00
author: Daniel
categories:
- [ Programming, .NET, F# ]
- [ Databases, LiteDB ]
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

_NOTE: This is the fourth post in a series; see [the introduction][intro] for information on requirements and links to other posts in the series._

myPrayerJournal v1 used [PostgreSQL][] with [Entity Framework Core][ef-core] for its backing store (which had [a stop on the v1 tour][v1-data]). v2 used [RavenDB][], and while I didn't write a tour of it, you can [see the data access logic][v2-data] if you'd like. Let's take a look at the technology we used for v3.

## About LiteDB

[LiteDB][] is a single-file, in-process database, similar to [SQLite][]. It uses a document model for its data store, storing Plain Old <abbr title="Common Language Runtime">CLR</abbr> Objects (POCOs) as Binary JSON (BSON) documents in its file. It support cross-collection references, customizable mappings, different access modes, and transactions. It allows documents to be queried via <abbr title="Language Integrated Query">LINQ</abbr> syntax, or via its own SQL-like language.

As I mentioned in the introduction, I picked it up for another project, and really enjoyed the experience. Its configuration could not be easier (the connection string is literally a path and file name), and it had good performance as well. The way it locks its database file, I can copy it while the application is up, which is great for backups. It was definitely a good choice for this project.

## The Domain Model

When I converted to RavenDB, the data structure ended up with one document per request; the history log and notes were stored as F# lists (arrays in JSON) within that single document. RavenDB supports indexes which can hold calculated values, so I had made an index that had the latest request text, and the latest time an action was taken on a request; when I displayed any list of requests, I queried the index, and got the calculated fields for free.

The model for v3 is very similar.

```fsharp
/// Request is the identifying record for a prayer request
[<CLIMutable; NoComparison; NoEquality>]
type Request = {
  /// The ID of the request
  id           : RequestId
  /// The time this request was initially entered
  enteredOn    : Instant
  /// The ID of the user to whom this request belongs ("sub" from the JWT)
  userId       : UserId
  /// The time at which this request should reappear in the user's journal by manual user choice
  snoozedUntil : Instant
  /// The time at which this request should reappear in the user's journal by recurrence
  showAfter    : Instant
  /// The type of recurrence for this request
  recurType    : Recurrence
  /// How many of the recurrence intervals should occur between appearances in the journal
  recurCount   : int16
  /// The history entries for this request
  history      : History list
  /// The notes for this request
  notes        : Note list
  }
```

_`History` has an "as-of" date/time, an action that was taken, and an optional request text field; `Note` has the same thing, minus the action._

## Customizing the POCO Mapping

### F#'s Special Types

If you look at the types in that list above, you'll spot exactly one primitive data type (`int16`). `Instant` comes from [NodaTime][], but the remainder are custom types. F# supports discriminated unions (DUs), which can be used in different ways to make invalid states unrepresentable. One way of doing this is via the single-case DU:

```fsharp
/// The identifier of a user (the "sub" part of the JWT)
type UserId =
  | UserId of string
```

Requests are associated with the user, via the `sub` field in the <abbr title="JSON Web Token">JWT</abbr> received from Auth0. That field is a string; but, in the handler that retrieves this from the `Authorization` header, it is returned as `UserId [sub-value]`. In this way, that string cannot be confused with any other string (such as a note, or a prayer request). Another way DUs can be used is to generate enum-like types, where each item is its own type:

```fsharp
/// How frequently a request should reappear after it is marked "Prayed"
type Recurrence =
  | Immediate
  | Hours
  | Days
  | Weeks
```

Here, these four values will refer to a recurrence, and it will take no others. This barely scratches the surface on DUs, but it should give you enough familiarity with them so that the rest of this makes sense.

> For the F#-fluent - you may be asking "Why didn't he define this with `Hours of int16`, `Days of int16`, etc. instead of putting the number separate from the type?" The answer is a combination of evolution - this is the way it worked in v1 - and convenience. I very well could have done it that way, and probably should at some point.
 
### Converting These Types in myPrayerJournal v2

F# does an excellent job of representing these types as CLR types; however, when they are serialized using the normal reflection-based serializers, the normally-transparent properties show up in the output. RavenDB (and Giraffe, when v1 was developed) uses [JSON.NET][] for its serialization, so it was easy to write a converter for the `UserId` type:

```fsharp
  /// JSON converter for user IDs
  type UserIdJsonConverter () =
    inherit JsonConverter<UserId> ()
    override __.WriteJson(writer : JsonWriter, value : UserId, _ : JsonSerializer) =
      (UserId.toString >> writer.WriteValue) value
    override __.ReadJson(reader: JsonReader, _ : Type, _ : UserId, _ : bool, _ : JsonSerializer) =
      (string >> UserId) reader.Value
```

Without this converter, a property "x", with a user ID value of "abc", would be serialized as:

```json
{ "x": { "Case": "UserId", "Value": "abc" } }
```

With this converter, though, the same structure would be:

```json
{ "x": "abc" }
```

For a database where you are querying on a value, or a JSON-consuming front end web framework, the latter is definitely what you want.

### Converting These Types in myPrayerJournal v3

With all of the above being said - LiteDB does not use JSON.NET; it uses its own custom `BsonMapper` class. This means that the conversions for these types would need to change. LiteDB does support creating mappings for custom types, though, so this task looked to be a simple conversion task. As I got into it, though, I realized that nearly every field I was using needed some type of conversion. So, rather than create converters for each different type, I created one for the document as a whole.

It was surprisingly straightforward, once I figured out the types! Here are the functions to convert the request type to its BSON equivalent, and back:


```fsharp
  /// Map a request to its BSON representation
  let requestToBson req : BsonValue =
    let doc = BsonDocument ()
    doc["_id"]          <- RequestId.toString req.id
    doc["enteredOn"]    <- req.enteredOn.ToUnixTimeMilliseconds ()
    doc["userId"]       <- UserId.toString req.userId
    doc["snoozedUntil"] <- req.snoozedUntil.ToUnixTimeMilliseconds ()
    doc["showAfter"]    <- req.showAfter.ToUnixTimeMilliseconds ()
    doc["recurType"]    <- Recurrence.toString req.recurType
    doc["recurCount"]   <- BsonValue req.recurCount
    doc["history"]      <- BsonArray (req.history |> List.map historyToBson |> Seq.ofList)
    doc["notes"]        <- BsonArray (req.notes   |> List.map noteToBson    |> Seq.ofList)
    upcast doc
  
  /// Map a BSON document to a request
  let requestFromBson (doc : BsonValue) =
    { id           = RequestId.ofString doc["_id"].AsString
      enteredOn    = Instant.FromUnixTimeMilliseconds doc["enteredOn"].AsInt64
      userId       = UserId doc["userId"].AsString
      snoozedUntil = Instant.FromUnixTimeMilliseconds doc["snoozedUntil"].AsInt64
      showAfter    = Instant.FromUnixTimeMilliseconds doc["showAfter"].AsInt64
      recurType    = Recurrence.ofString doc["recurType"].AsString
      recurCount   = int16 doc["recurCount"].AsInt32
      history      = doc["history"].AsArray |> Seq.map historyFromBson |> List.ofSeq
      notes        = doc["notes"].AsArray   |> Seq.map noteFromBson    |> List.ofSeq
      }
```

The downside to this technique is // TODO stopped here 


[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[PostgreSQL]: https://www.postgresql.org "PostgreSQL"
[ef-core]: https://docs.microsoft.com/en-us/ef/core/ "Overview of Entity Framework Core | Microsoft Docs"
[v1-tour]: /2018/a-tour-of-myprayerjournal/the-data-store.html "A Tour of myPrayerJournal: The Data Store | The Bit Badger Blog"
[RavenDB]: https://ravendb.net "RavenDB"
[v2-data]: https://github.com/bit-badger/myPrayerJournal/blob/2.2/src/MyPrayerJournal.Api/Data.fs "myPrayerJournal v2 data access module"
[LiteDB]: https://www.litedb.org "LiteDB"
[SQLite]: https://sqlite.org "SQLite"
[NodaTime]: https://nodatime.org "NodaTime"
[JSON.NET]: https://www.newtonsoft.com/json "JSON.NET"
[lite-map]: https://www.litedb.org/docs/object-mapping/ "Object Mapping - LiteDB"

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
