---
layout: post
title: "A Tour of myPrayerJournal v3: The Data Store"
date: 2021-12-01 17:33:00
author: Daniel
categories:
- [ Programming, .NET, F# ]
- [ Databases, LiteDB ]
- [ Projects, myPrayerJournal ]
- [ Series, A Tour of myPrayerJournal v3 ]
tags:
- async
- bson
- discriminated union
- ef core
- f#
- giraffe
- json
- linq
- litedb
- mapping
- postgresql
- ravendb
---

_NOTE: This is the fourth post in a series; see [the introduction][intro] for information on requirements and links to other posts in the series._

myPrayerJournal v1 used [PostgreSQL][] with [Entity Framework Core][ef-core] for its backing store (which had [a stop on the v1 tour][v1-data]). v2 used [RavenDB][], and while I didn't write a tour of it, you can [see the data access logic][v2-data] if you'd like. Let's take a look at the technology we used for v3.

## About LiteDB

[LiteDB][] is a single-file, in-process database, similar to [SQLite][]. It uses a document model for its data store, storing Plain Old <abbr title="Common Language Runtime">CLR</abbr> Objects (POCOs) as Binary JSON (BSON) documents in its file. It supports cross-collection references, customizable mappings, different access modes, and transactions. It allows documents to be queried via <abbr title="Language Integrated Query">LINQ</abbr> syntax or via its own SQL-like language.

As I mentioned in the introduction, I picked it up for another project, and really enjoyed the experience. Its configuration could not be easier -- the connection string is _literally_ a path and file name -- and it had good performance as well. The way it locks its database file, I can copy it while the application is up, which is great for backups. It was definitely a good choice for this project.

## The Domain Model

When I converted from PostgreSQL to RavenDB, the data structure ended up with one document per request; the history log and notes were stored as F# lists (arrays in JSON) within that single document. RavenDB supports indexes which can hold calculated values, so I had made an index that had the latest request text, and the latest time an action was taken on a request. When v2 displayed any list of requests, I queried the index, and got the calculated fields for free.

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

A few notes would probably be good here:

- The `CLIMutable` attribute allows this non-nullable record type to be null, and generates a zero-argument constructor that reflection-based processes can use to create an instance. Both of these are needed to interface with a C#-oriented data layer.
- By default, F# creates comparison and equality implementations for record types. This type, though, is a simple data transfer object, so the `NoEquality` and `NoComparison` attributes prevent these from being generated.
- Though not shown here, `History` has an "as-of" date/time, an action that was taken, and an optional request text field; `Note` has the same thing, minus the action but requiring the text field.

## Customizing the POCO Mapping

If you look at the fields in the `Request` type above, you'll spot exactly one primitive data type (`int16`). `Instant` comes from [NodaTime][], but the remainder are custom types. These are POCOs, but not your typical POCOs; by tweaking the mappings, we can get a much more efficient BSON representation.

### Discriminated Unions

F# supports discriminated unions (DUs), which can be used in different ways to construct a domain model in such a way that an invalid state cannot be represented (TL;DR - "make invalid states unrepresentable"). One way of doing this is via the single-case DU:

```fsharp
/// The identifier of a user (the "sub" part of the JWT)
type UserId =
  | UserId of string
```

Requests are associated with the user, via the `sub` field in the <abbr title="JSON Web Token">JWT</abbr> received from [Auth0][]. That field is a string; but, in the handler that retrieves this from the `Authorization` header, it is returned as `UserId [sub-value]`. In this way, that string cannot be confused with any other string (such as a note, or a prayer request).

Another way DUs can be used is to generate enum-like types, where each item is its own type:

```fsharp
/// How frequently a request should reappear after it is marked "Prayed"
type Recurrence =
  | Immediate
  | Hours
  | Days
  | Weeks
```

Here, these four values will refer to a recurrence, and it will take no others. This barely scratches the surface on DUs, but it should give you enough familiarity with them so that the rest of this makes sense.

> _For the F#-fluent - you may be asking "Why didn't he define this with `Hours of int16`, `Days of int16`, etc. instead of putting the number in `Request` separate from the type?" The answer is a combination of evolution -- this is the way it worked in v1 -- and convenience. I very well could have done it that way, and probably should at some point._
 
### Converting These Types in myPrayerJournal v2

F# does an excellent job of transparently representing DUs, `Option` types, and others to F# code, while their underlying implementation is a CLR type; however, when they are serialized using traditional reflection-based serializers, the normally-transparent properties appear in the output. RavenDB (and Giraffe, when v1 was developed) uses [JSON.NET][] for its serialization, so it was easy to write a converter for the `UserId` type:

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

With all of the above being said -- LiteDB does not use JSON.NET; it uses its own custom `BsonMapper` class. This means that the conversions for these types would need to change. LiteDB does support creating mappings for custom types, though, so this task looked to be a simple conversion task. As I got into it, though, I realized that nearly every field I was using needed some type of conversion. So, rather than create converters for each different type, I created one for the document as a whole.

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

Each of these round-trips as the same value; line 6 (`doc["userId"]`) stores the string representation of the user ID, while line 19 (`userId =`) creates a strongly-typed `UserId` from the string stored in database.

> _The downside to this technique is that LINQ won't work; passing a `UserId` would look for the default serialized version, not the simplified string version. This is not a show-stopper, though, especially for such a small application as this. If I had wanted to use LINQ for queries, I would have written several type-specific converters instead._ 

## Querying the Data

In v2, there were two different types; `Request` was what was stored in the database, and `JournalRequest` was the type that included the calculated fields included in the index. This conversion came into the application; `ofRequestFull` is a function that performs the calculations, and returns an item which has full history and notes, while `ofRequestLite` does the same thing without the history and notes lists.

With that knowledge, here is the function that retrieves the user's current journal:

```fsharp
/// Retrieve the user's current journal
let journalByUserId userId (db : LiteDatabase) = backgroundTask {
  let! jrnl = db.requests.Find (Query.EQ ("userId", UserId.toString userId)) |> toListAsync
  return
    jrnl
    |> Seq.map JournalRequest.ofRequestLite
    |> Seq.filter (fun it -> it.lastStatus <> Answered)
    |> Seq.sortBy (fun it -> it.asOf)
    |> List.ofSeq
  }
```

Line 3 contains the LiteDB query; when it is done, `jrnl` has the type `System.Collections.Generic.List<Request>`. This "list" is different than an F# list; it is a concrete, doubly-linked list. F# lists are immutable, recursive item/tail pairs, so F# views the former as a form of sequence (as it extends `IEnumerable<T>`). Thus, the `Seq` module calls in the return statement are the appropriate ones to use. They execute lazily, so filters should appear as early as possible; this reduces the number of latter transformations that may need to occur.

Looking at this example, if we were to sort first, the entire sequence would need to be sorted. Then, when we filter out the requests that are answered, we would remove items from that sequence. With sorting last, we only have to address the full sequence once, and we are sorting a (theoretically) smaller number of items. Conversely, we do have to run the `map` on the original sequence, as `lastStatus` is one of the calculated fields in the object created by `ofRequestLite`. Sometimes you can filter early, sometimes you cannot.

_(Is this micro-optimizing? Maybe; but, in my experience, taking a few minutes to think through collection pipeline ordering is a lot easier than trying to figure out why (or where) one starts to bog down. Following good design principles isn't premature optimization, <abbr title="In My Opinion">IMO</abbr>.)_

## Getting a Database Connection

The example in the previous section has a final parameter of `(db: LiteDatabase)`. As Giraffe sits atop ASP.NET Core, myPrayerJournal uses the traditional dependency injection (DI) container. Here is how it is configured:

```fsharp
/// Configure dependency injection
let services (bldr : WebApplicationBuilder) =
  // ...
  let db = new LiteDatabase (bldr.Configuration.GetConnectionString "db")
  Data.Startup.ensureDb db
  bldr.Services
    // ...
    .AddSingleton<LiteDatabase> db
  |> ignore
  // ...
```

The connection string comes from `appsettings.json`. `Data.Startup.ensureDb` makes sure that requests are indexed by user ID, as that is the parameter by which request lists are queried; this also registers the converter functions discussed above. LiteDB has an option to open the file for shared access or exclusive access; this implementation opens it for exclusive access, so we can register that connection as a singleton. (LiteDB handles concurrent queries itself.)

Getting the database instance out of DI is, again, a standard Giraffe technique:

```fsharp
/// Get the LiteDB database
let db (ctx : HttpContext) = ctx.GetService<LiteDatabase> ()
```

This can be called in any request handler; here is the handler that displays the journal cards:

```fsharp
// GET /components/journal-items
let journalItems : HttpHandler =
  requiresAuthentication Error.notAuthorized
  >=> fun next ctx -> backgroundTask {
    let  now   = now ctx
    let! jrnl  = Data.journalByUserId (userId ctx) (db ctx)
    let  shown = jrnl |> List.filter (fun it -> now > it.snoozedUntil && now > it.showAfter)
    return! renderComponent [ Views.Journal.journalItems now shown ] next ctx
    }
```

## Making LiteDB Async

I found it curious that LiteDB's data access methods do not have async equivalents (ones that would return `Task<T>` instead of just `T`). My supposition is that this is a case of <abbr title="You Ain&apos;t Gonna Need It">YAGNI</abbr>. LiteDB maintains a log file, and makes writes to that first; then, when it's not busy, it synchronizes the log to the file it uses for its database. However, I wanted to control when that occurs, and the rest of the request/function pipelines are async, so I set about making async wrappers for the applicable function calls.

Here are the data retrieval functions:

```fsharp
/// Convert a sequence to a list asynchronously (used for LiteDB IO)
let toListAsync<'T> (q : 'T seq) =
  (q.ToList >> Task.FromResult) ()

/// Convert a sequence to a list asynchronously (used for LiteDB IO)
let firstAsync<'T> (q : 'T seq) =
  q.FirstOrDefault () |> Task.FromResult

/// Async wrapper around a request update
let doUpdate (db : LiteDatabase) (req : Request) =
  db.requests.Update req |> ignore
  Task.CompletedTask
```

And, for the log synchronization, an extension method on `LiteDatabase`:

```fsharp
/// Extensions on the LiteDatabase class
type LiteDatabase with
  // ...
  /// Async version of the checkpoint command (flushes log)
  member this.saveChanges () =
    this.Checkpoint ()
    Task.CompletedTask
```

None of these actually make the underlying library use async I/O; however, they do let the application's main thread yield until the I/O is done. Also, despite the `saveChanges` name, this is **not required** to save data into LiteDB; it is there once the insert or update is done (or, optionally, when the transaction is committed). 

## Final Thoughts

As I draft this, this paragraph is on line 280 of this post's source; the entire [Data.fs][] file is 209 lines, including blank lines and comments. The above is a moderately long-winded explanation of what is nicely terse code. If I had used traditional C#-style POCOs, the code would likely have been shorter still. The backup of the LiteDB file is right at half the size of the equivalent RavenDB backup, so the POCO-to-BSON mapping paid off there. I'm quite pleased with the outcome of using LiteDB for this project.

Our [final stop on the tour][part4] will wrap up with overall lessons learned on the project.


[intro]: /2021/a-tour-of-myprayerjournal-v3/introduction.html "A Tour of myPrayerJournal v3: Introduction | The Bit Badger Blog"
[PostgreSQL]: https://www.postgresql.org "PostgreSQL"
[ef-core]: https://docs.microsoft.com/en-us/ef/core/ "Overview of Entity Framework Core | Microsoft Docs"
[v1-data]: /2018/a-tour-of-myprayerjournal/the-data-store.html "A Tour of myPrayerJournal: The Data Store | The Bit Badger Blog"
[RavenDB]: https://ravendb.net "RavenDB"
[v2-data]: https://github.com/bit-badger/myPrayerJournal/blob/2.2/src/MyPrayerJournal.Api/Data.fs "myPrayerJournal v2 data access module"
[LiteDB]: https://www.litedb.org "LiteDB"
[SQLite]: https://sqlite.org "SQLite"
[NodaTime]: https://nodatime.org "NodaTime"
[Auth0]: https://auth0.com "Auth0"
[JSON.NET]: https://www.newtonsoft.com/json "JSON.NET"
[lite-map]: https://www.litedb.org/docs/object-mapping/ "Object Mapping - LiteDB"
[Data.fs]: https://github.com/bit-badger/myPrayerJournal/blob/3/src/MyPrayerJournal/Data.fs "myPrayerJournal v3 Data.fs file"
[part4]: /2021/a-tour-of-myprayerjournal-v3/conclusion.html "A Tour of myPrayerJournal v3: Conclusion | The Bit Badger Blog"
