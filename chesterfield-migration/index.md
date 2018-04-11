---
title: Chesterfield Migration
layout: default
version: 1.0.0-rc1
publishedAt: Wed 11. Apr 13:16:31 CEST 2018
# in vim `:r! date`
lastUpdatedAt: Wed 11. Apr 13:16:54 CEST 2018
permalink: /chesterfield-migration/
---

# [Sync<br/>Tank](/) Chesterfield Migration

## An eager server-side multi-version distributed migration strategy
{:.no_toc}

published {{ page.publishedAt | date: '%B %d, %Y' }}, last updated {{ page.lastUpdatedAt | date: '%B %d, %Y' }}

## Table of contents
{:.no_toc}

1. this unordered seed list will be replaced by toc as unordered list
{:toc}

## Introduction

Migrating data is not trivial. Migrating distributed data that is needed in lots of places at the same time can quickly become a formidable challenge. In a [recent article](/distributed-migration-strategies/), we have addressed many of the complications that schema migrations in distributed systems may face. In this article, we want to go beyond the problem description and start discussing potential solutions.

The strategy we are presenting here is an *eager server-side multi-version migration*. If you are not familiar with this terminology, you may want to revisit the [migration taxonomy](/distributed-migration-strategies/#a-taxonomy-for-reasoning-about-migrations) we have introduced previously. But if you decide to read on, rest assured that we will cover all aspects necessary to follow the discussion in this article.

In what follows, we will first build up a sufficiently complex use case for distributed migrations based on a todo application. This article is not a tutorial, but we feel that a good example will help us bring across some points more concisely. Next, we will talk about CouchDB. This is our database of choice and we want to take a closer look at how it supports what we want to do. In particular, we need to think about how to design the schema so that it can support the operations we want to perform. With these preliminaries out of the way, we will then invite you to a detailled technical discussion about *transformers*, the main work horse of our migration strategy, before concluding our discussion with a summary of what we have learned.

<figure>
  <img src="/chesterfield-migration/images/chesterfield.jpg" alt="The authors: Matthias and Johannes" />
  <figcaption>
    <b>Chesterfield.</b>
    <span>
      A type of luxurious couch. And an appropriate name for a migration strategy that has you covered in all kinds of circumstances.
    </span>
  </figcaption>
</figure>

In order to have a memorable reference, we are going to brand our strategy as *chesterfield migration*. A chesterfield is a type of luxurious couch, and we feel this is an appropriate term as we are going to describe an approach that meets all kinds of challenging requirements comfortably – and is based upon CouchDB.


## Revisiting an example application

We want to build large systems featuring offline-capable applications and clients on multiple platforms. We want to support clients that require different schema versions of application data. We want to provide backwards-compatibility while staying agile. Because it is easy to get lost in a complex scenario like this, we would like to describe an example use case that brings all these requirements together. In this section, you will find a concise summary of the [example todo application](/distributed-migration-strategies/#setting-the-stage-a-toy-problem) we have designed in our introductory article. Feel free to skip this section if the setup is fresh in your memory, or read on for a quick refresher of how the schema evolved across the different versions of the app.

### Version One: Starting Simple

In the beginning, there are only todos. We start with a `db-per-user` setup where users have all their todo items in their own database. CouchDB mananges data synchronization across multiple clients, and the schema only comprises the most basic features every todo item exhibits. In particular, each todo item is required to have a `title` and an `isDone` property. This is version `1.0.0` of the schema which only consists of version `1.0.0` of todo item documents.

By popular request we decide to add an optional `isImportant` property to todo items, and we also introduce a `settings` document to allow users to switch between different color themes. These changes can be accomodated into the schema without breaking existing behavior, so they are two new *features*. We can release these new versions, and new app versions that work with them, without having to migrate existing data at all. At this point, our schema manifest for version `1.1.0` looks like this:

```json
{
  "name": "todo-app-schema",
  "version": "1.1.0",
  "dependencies": {
    "settings": "^1.0.0",
    "todo-item": "^1.1.0"
  }
}
```

Later on, when we discuss the more intricate details of how to migrate between different document versions, we will need concrete examples. In preparation for this discussion, the following listings show a full todo item and settings document respectively.


```json
{
  "_id": "todo-item@1:ab8ee6240d8b5143884127d3408b1a85",
  "schema": "todo-item",
  "version": 1,
  "title": "Compare apples to oranges",
  "isDone": true,
  "isImportant": true,
  "createdAt": "2018-04-01T00:00:00.000Z"
}
```

```json
{
  "_id": "settings@1",
  "schema": "settings",
  "version": 1,
  "color": "#e20074"
}
```

At this point, our todo application is out on the market. Data schema `1.1.0` supports various versions of iOS and Android apps, webapps, and service workers that form a big software ecosystem. Some of these clients have offline-capabilities, which implies that they are not easily accessible for maintenance and might miss updates. So far, this is not a huge problem as far as the schema is concerned. Unfortunately, we are just about to introduce a breaking schema change.

### Version Two: Tracking Item Status

Because a simple `isDone` flag is not very expressive, we decide to replace it with a `status` property that can have one of many states. Now todo items can be `active`, `blocked`, `halted`, `done`, etc. To account for this change, we have to update the schema definition of todo item documents. The required `isDone` property is replaced with a required `status`. Let's take a look at the manifest for the updated schema.

```json
{
  "name": "todo-app-schema",
  "version": "2.0.0",
  "dependencies": {
    "settings": "^1.0.0",
    "todo-item": "^2.0.0"
  }
}
```

As you can see, we had to increase the major version of the todo item schema - and hence the major version of the whole schema. Replacing `isDone` with the `status` is a breaking change for two reasons. First of all, existing apps are expecting documents to have the `isDone` property, so they will not be able to handle documents of the new format. And secondly, once we have built apps that can work with the new documents, these apps won't be able to read the existing documents. This requires us migrate old documents up to the new format - and to migrate documents of the new format down to the old one if we want to maintain backwards compatibility. Before moving on, here is the todo item from above in it's new, updated version. We have mapped the old `isDone: true` to a `status` of `done`.

```json
{
  "_id": "todo-item@2:ab8ee6240d8b5143884127d3408b1a85",
  "schema": "todo-item",
  "version": 2,
  "title": "Compare apples to oranges",
  "status": "done",
  "isImportant": true,
  "createdAt": "2018-04-01T00:00:00.000Z"
}
```

### Version Three: Grouping Items

The final feature we will discuss is sorting todo items into groups. To this end, we introduce a new document type, a `group` document, and require todo items to have a `groupId` property that links them to their respective group. This brings us to version `3.0.0` of our schema.

```json
{
  "name": "todo-app-schema",
  "version": "3.0.0",
  "dependencies": {
    "group": "^1.0.0",
    "settings": "^1.0.0",
    "todo-item": "^3.0.0"
  }
}
```

Introducing a new document type on its own does not constitute a breaking change as we have seen when adding the `settings` document above. The reason is that no application can *require* a document to be present - documents may always come in late in the replication process. Apps must always be prepared to deal with missing documents.

Adding another required property, however, *is* a breaking schema change, just like adding the required `status` was. Once again, we need to increase the major version number of the todo item schema after adding the `groupId`. As consequence, the whole schema's version must be increased. The following listings show the updated todo item with its new group.

```json
{
  "_id": "todo-item@3:ab8ee6240d8b5143884127d3408b1a85",
  "schema": "todo-item",
  "version": 3,
  "title": "Compare apples to oranges",
  "status": "done",
  "isImportant": true,
  "groupId": "e302f183a96194f7d19dce0eaf5e3cf8",
  "createdAt": "2018-04-01T00:00:00.000Z"
}
```

```json
{
  "_id": "group@1:e302f183a96194f7d19dce0eaf5e3cf8",
  "schema": "group",
  "version": 1,
  "name": "Things I want to do before retiring"
}
```

These are the three major schema versions that we have to maintain in parallel. In order to keep all versions up to date when one piece of data changes it will be necessary to update existing documents in multiple versions. For instance, if a user creates a todo item with their very old app that still runs version one, we need to migrate documents up so that the new todo is not lost on devices that run newer versions. Similarly, we need to migrate changes made with newer apps down to support older apps. Apart from the actual migration logic, this setup requires some additional infrastructure to work well. First, we will need to distribute documents with multiple versions throughout the system, and then we will need to manage documents with multiple versions from inside the applications. Let's take a closer look at these two tasks.


## Handling Distributed Multi-Version Data

You know by now that the chesterfield migration is an *eager server-side multi-version migration*. That is quite a mouthful, so let's break it down and see what it means in practice.

* **Server-side** means that the transformation of documents happens on the server-side and is not handled by the clients. This requires us to set up a *backend service* responsible for performing the migration logic. In our case, the work will be done by what we call *transformer modules*.
* **Eager** refers to the fact that migrations are performed as soon as documents are available. To this end, we will listen to CouchDB's `_changes`-endpoint and react when new document update information is coming in.
* **Multi-version** means that multiple document versions will be maintained in parallel, allowing us to support multiple application versions at the same time.

[Figure 3](#figure-3) illustrates the idea in broad strokes: when necessary, transformers create multiple versions of documents so that a single shared database can support multiple app versions. From the backend-database, updated documents are then being replicated to the clients.

<figure class="diagram" id="figure-3">
  <img src="/chesterfield-migration/images/per-version-docs.svg" alt="Schematic view of chesterfield migration" />
  <figcaption>
    <b>Figure 3: Chesterfield migration.</b>
    <span>
      Documents exist in multiple versions in order to support multiple application versions through a single database. Transformers are the engines that perform up and down migrations.
    </span>
  </figcaption>
</figure>

In the next section, we will give our undivided attention to transformer modules. But before that, let us spend some time thinking through the *multi-version* aspect. Two questions in particular will concern us in the following: How can we control which version of the data gets distributed to which part of the system? And how can we manage multiple versions of the same documents in a single database?


### Replication channels

We want to maintain multiple versions of the same data in parallel because we would like to provide backwards compatibility for older applications. This is why we migrate data on the server and distribute updates across the system. If we dig a little further though, we will find that not *all* document versions will have to be propagated to *all* clients. For instance, if a new client produces a todo item according to version three, and if the server migrates this document down to versions one and two, the newer client would not be interested in receiving the older documents. We can save a lot of traffic and storage if we can prevent newer clients from receiving older documents.

Do we also need to prevent clients from receiving *newer* document versions than they can currently handle? The answer depends on which side we favor in a trade-off between replication traffic and ease of application update. On the one hand, preventing newer documents from being replicated to clients that are not yet ready results in lower traffic. On the other hand, the newer documents will have to be replicated anyway once the application is updated and is expecting newer schema versions.

And there is more: if we allow newer documents to reach clients that are not yet ready we can support what we would like to call *seamless migrations*, migrations without downtime. This is possible because we could migrate documents up to a new version, distribute them across the system, and only provide application updates once the clients had enough time to receive the new data. Once the apps get updated, they find all the new data they need is already in their local database!

Our short discussion has elicited two new requirements with respect to replication management:

1. Do not replicate older documents to newer clients. They don't need those anymore.
2. Replicate newer documents to older clients. They can use those once they get updated.

Replication flow management is realised in CouchDB through [filtered replications](http://docs.couchdb.org/en/latest/replication/intro.html#controlling-which-documents-to-replicate). The traditional way to specify which data to replicate is through filter functions. However, as of CouchDB 2.0.0 there is a new and very performant way to implement filtered replications with the help of [Mango selectors](http://docs.couchdb.org/en/latest/api/database/find.html#find-selectors). Let's see them in action.

The following listing shows a selector that would be used by todo apps of version two to replicate documents from the server-side database. Note how the selector allows for specifying boolean operations on document attributes via `$not` and `$lt`.

```json
{
  "selector": {
    "$not": {
      "schema": "todo-item",
      "version": { "$lt": 2 }
    }
  }
}
```

There are a few noteworthy aspects about this selector. We said it's supposed to be used by clients that work with schema version two. Recall from the example manifests that schema version two requires todo items of version two. But since we want apps to receive all newer documents as well we ask to exclude only lower versions from the replication. Similarly, this selector will also replicate all documents that are not todo item documents. This includes group and settings documents of version one and higher as well as all other types that may be added in the future. This way, clients can stay open to new schema versions and be prepared for the next seamless migration.

[maybe an image here that illustrates how the selector works?]

As a last point we'd like to reiterate that replications take time and may lead to temporarily incomplete data. For instance, the todo item may have already been replicated while the corresponding group document is still missing. But this is a general learning about CouchDB: clients have to be able to deal with this kind of incomplete data anyway. If you really need pieces of data to be present together consider keeping them together in one document.

### Multiple schema versions in a single database

After successful replication clients will have all necessary data in their local databases, although potentially in multiple versions. This raises the question of how parallel schema versions can be managed within a single database.

An obvious first requirement is that every document's `_id` now has to reflect the version number. This was already addressed when we introduced the migration taxonomy in our previous article. *Single-version* and *multi-db* migrations do not have to modify document identifiers - they can simply reuse them across versions. *Multi-version* migrations, however, cannot afford this convenience. One option for reflecting the version in the `_id` is to add it right after the schema title. This would, for instance, create todo items with `_id`s like `todo-item@2:ce2f71ee7db9d6decfe459ca9d000df5`. A transformator performing an up-migration of this document would accordingly have to increase the version number to 3 without modifying the hash so the document could still be identified.

All the example documents in the previous section aleady carry their version in the `_id`. The astute reader may have noticed, however, that these are only the major numbers of the document versions. If we are using semantic versioning for schemas, why does it say `todo-item@2:<uuid>` instead of `todo-item@2.0.0:<uuid>`? The reason for this is that any application that can work with version `2.0.0` can also work with version `2.1.0` or `2.13.5`. If it couldn't, there would have been a breaking change in the schema and we would have had to move to version `3` at some point. In this sense, all documents of the same major version can be treated as belonging to the same collection. Hence, there is no need to be overly specific with the `_id`.

As another aside, be careful not to rely on document versions when it comes to imlementing associations. If we associate, say, an address with a user through a `userId`-attribute on the address, this should not specify the version of the user document. Otherwise migrations could destroy existing associations.

Apart from reflecting the schema version in the document identifier we have made sure from early on to keep `schema` and `version`-attributes in each document. This will pay off now as clients need to retrieve exactly the version of a piece of data that they can currently work with. Traditionally most data was retrieved from CouchDB via *views* but once again Mango selectors are a recent, very performant alternative. By way of an example consider a client working with data schema version two that wants to display a list of todos. The same todo item may exist in multiple versions in the database, but the client can retrieve exactly the version it knows how to handle using the following selector.

```json
{
  "selector": {
    "schema": "todo-item",
    "version": 2
  },

  "sort": [{
    "createdAt": "desc"
  }]
}
```

These three guiding principles - reflecting the document version in the identifier, establishing associations through raw ids, and querying documents by relevant `schema` and `version` - allow  clients to handle multiple parallel schema versions in a single database. All this is possible without having to change any URLs when communicating with the server-side database.

Before moving on to see how the transformer engine works under the hood we would like to address a concern you may have regarding the size of local databases. Memory is still a somewhat scarce resource especially for mobile clients. Would the approach we have sketched out here not bloat client databases unnecessarily by keeping legacy schema versions around when they are in fact no longer needed? As user data builds up, this may indeed become a problem at some point. If that's the case, don't despair, as there are ways to perform some data compaction. We suggest creating an additional local database on the client and running a filtered replication that replicates only relevant document versions. After that, switch over to the new database and discard the old one alongside all legacy schema versions.


## Transformer modules

We are finally ready to take a close look at transformer modules, the main workhorse of our migration strategy. Transformer modules, or transformers for short, live on the server, they are responsible for noticing when a document has to be migrated and they create and persist new document versions. These are three different tasks - listening, creating, and persisting - and our transformers will be composed of three corresponding components. To be more precise, each transformer module will consist of a *registry*, a *migration logic unit*, and an *update handler*. Let's introduce each of those in turn and see what considerations have led us to design them the way we did.

### The registry

Transformers are supposed to migrate documents once they are replicated to the server-side database. To figure out when to get active, transformers can inspect CouchDB's `_changes`-endpoint (either directly or via a helpful service that creates a changes feed for this purpose) to get a rough idea of what's happening on the database. In particular, they can find out which document ids have been involved in any database event. Because we designed document ids to reflect the document schema transformers have a way of knowing when to perform a migration: every time a database event is happening, a transformer can check if the document involved is of the type and version it is responsible for. And how do transformers know which documents are their responsibility? That's what the registry is for.

The general idea is easy enough, but designing an understandable and maintainable system of transformer-responsiblities will be more challenging. If you remember from our discussion above, the fact that *Chesterfield* is a *multi-version* migration means that we will need to migrate documents between all possible (currently supported) versions. Every `todo-item-1` needs to be migrated to every other supported version and every other supported version needs to be migrated to `todo-item-1` - on every document creation and every document update. To manage the upcoming complexity inherent in this task we suggest to begin by introducing *lean* transformers that only perform the most basic of migration tasks: changing a single document type from one version to another. For instance, there might be a dedicated transformer just to change `todo-item-2` documents to `todo-item-1` documents.

This simple migration task does not look like its posing much of a challenge in terms of implementation, but once we start thinking through the moving parts, things are not so simple anymore. We asked you to be prepared for a more technical discussion, and this is one of the points where we believe a detailed treatment can get you started off on the right track and save you a lot of time and headache in the long run. So let's see what's so tricky about updating `todo-item-1` to `todo-item-2` documents.

To approach this problem, we would like to start with the intuition we had when we were beginning to design the first transformers. At the time we thought that a transformer should listen on the changes feed for the *source document* it is responsible for. For example, the  transformer that migrates todo items from version one to version two should get active every time a document of `todo-item-1` appears in the changes feed. According to our example data schema the transformer would then create the corresponding `todo-item-2` and `status-1` documents. So even in this simple example, a document of a single type is split into multiple documents of different types during the migration process. For the reverse migration this means that multiple documents will have to be merged into a single one, and this is where the naïve approach will start to break.

To illustrate the failure scenario, let's playback in *slow-motion*, as it were, what happens when Marten creates a new todo item with his Android app which runs on version *two* of the data schema. The action will create two documents, namely a `todo-item-2` and a `status-1` document. Once the documents are created, they are replicated to the server-side database where they arrive one by one (clearly visible thanks to our slow motion setting were network traffic is almost unbearably slow). Say the new `todo-item-2` document arrives first. This triggers a transformer to wake up which is responsible for migrating todo items down from version two to version one. To create version one of the document, however, the transformer needs to set the `isDone` state properly which it cannot know until the `status-1` document has been replicated. So *unless all relevant documents have been replicated, the migration has to be aborted*. When the `status-1` document comes in after a while, all information is now there, but the *source-oriented* todo item transformer will not feel responsible anymore and the whole migration process fails.

<figure class="diagram" id="figure-4">
  <img src="/chesterfield-migration/images/responsibilities.svg" alt="Schematic view of responsibility types" />
  <figcaption>
    <b>Figure 4: Source versus target-oriented responsibility design - a subtle but impactful change of perspective.</b>
    <span> On the left, a source-oriented transformer listens for one particular incoming schema and produces potentially multiple output documents. On the right, a target-oriented transformer tries to create one particular schema from potentially multiple relevant input documents.
    </span>
  </figcaption>
</figure>

The solution to this problem is a *target-oriented* responsibility design. Transformers should be written with respect to the document classes they are *outputting*, not the ones they are reading. Figure 4 illustrates the different approaches. For our example this would mean that our todo item transformer would not focus on *reading* `todo-item-2` documents but on *creating* `todo-item-1` documents. This looks like no more than a change of perspective, but it makes a huge difference for a correct implementation. The registry for this transformer would now contain all document types that might play a role for creating `todo-item-1` documents, namely `todo-item-2` as well as `status-1`. Everytime any of these documents arrive, the transformer comes alive and tries to create or update a `todo-item-1` document. Sometimes this process may fail as we just saw, if replication is still ongoing, but eventually all relevant documents will have been replicated and the migration can succeed. To be very clear about this new learning, let's summarize our considerations as a basic rule:

> The registry of a target-oriented transformer should have an entry for every document schema version that can contain relevant information for the documents the transformer is supposed to output.

The impact of this new perspective will become even clearer once we discuss the inner workings of the transformer's *migration logic unit* in the next section. But before we go there, now is a good time for a short aside on the number of transformers we will have to write and maintain in order for this approach to work.

#### Aside: a rough and ready complexity analysis

Migrating between all existing (or at least all supported) versions of a data schema might raise problems of maintainability through the sheer number of transformers necessary. A bit of accounting can help us get the discussion started. Say we start off with a single document type that is in version `v1`. We now update the schema to `v2`, so we will need to provide two transformers, one that generates `v1` documents from `v2` and one that generation `v2` documents from `v1`. After the next update to `v3` there are now already *six* transformers necessary to support conversion from every version to every other. In general, from every schema version that has ever existed in the system there may still be some documents around. So if we want to support all app versions we will have to provide as many transformers as there are ordered pairs, meaning $n (n - 1)$ transformers if the current data schema has version `n`. That's $n^2 - n$ - the amount of transformers grows quadratically with the number of supported schema versions! And we have only just considered a single document type.

A data schema in the wild can easily accommodate dozens of document types. In our example we just had three types (`todo-item`, `status`, `settings`) but to be more general let's say we have `t` different document types. If we introduce a new version for every type with every data schema update we need $t n (n - 1)$ transformers in total. If there are 6 schema versions and only 10 document types this already amounts to 300 transformers, which would be a nightmare to write, let alone to maintain.

```
- strategy:
  - drop version support
  - splitting data schema into smaller types, not updating complete data schema
  - transformer-chains (avoiding loops)
```






But for now it's time to move on to the second part of our transformers where the actual migration logic gets implemented.

### The migration logic unit

```
- schema knowledge: how to get it, what is needed
- splitting up documents
- merging documents
  - needs retrieval
    -> more views?
- caveat: the concept of document-level transactions which emphasizes the need to keep information that *must* be available together in a single document and otherwise embrace the idea that updates might only happen incrementally and might even lead to temporarily inconsistent data if not treated carefully.
```


### The update handler

```
- MLU outputs current document, but may create conflict
- fetch and save new docs
- how to avoid loops
  -> the problem of migration loops where a transformer might migrate a document up and persist it only for another migrator to pick it up and migrate it back down which might trigger the first transformer again and so on,
- conflict generation, inter-schema conflicts
```




```
Move this content to Appendix
  -> better approach: target-oriented
    -> problem: what happens when an *old version* is already there that is not yet updated,
        e.g. there is already a status-1 but it is not up to date...
- the registry lists all docs that are needed to create the *target*-version
- have one transformer per version-version pair and type
  -> complexity-analysis (rough)
    - naive:
      - (n-1)^2 version combinations
      - t document-types
      => O(t * n^2) transformers, e.g. with 5 versions and 20 types => 500 transformers!
```


```
- transactions are on doc level
- downsides:
  - not easy to drop (purge) old versions
```

## Summary and outlook

### We are not alone

The solution we discuss here does not come out of thin air. It has emerged from a longer discussion with contributers from different institutions and the offline-first community. To give credit where credit is due, we would like to begin with a very brief recap of the background of our approach.

During his time with eHealth Africa Johannes began to think about the problem of distributed migrations together with Jan Lehnardt. When he later joined immmr, the company was still on its way to developing a market-ready version of its first product. At this time, there were a lot of concerns about the viability of schema migrations. The only real alternative to migrations - getting everything right from the start - has some problems of its own so Johannes and Ben Kampmann generated several ideas for migration strategies, from which the approach we are going to present here emerged as a final result.

<figure>
  <img src="/chesterfield-migration/images/offline-camp-migration-session.jpg" alt="Migration session at Offline Camp 2017 Berlin" />
  <figcaption>Foto by Gregor Martinus: Migration session at Offline Camp 2017 Berlin</figcaption>
</figure>

The [offline camp berlin 2017](http://offlinefirst.org/camp/berlin/) provided an excellent opportunity to discuss the strategy with members of the offline-first community and we gratefully appreciate the thoughtful comments from Gregor Martinus, Bradley Holt, Martin Stadler (a former immmr-colleage!) and others. These discussions gave us a lot more confidence that we have found a robust approach that can persist through a number of challenging edge-cases.



## Older stuff from before, waiting for reuse...

#### Functionality duplication leads to unnecessary code

The first problem to address is one of complexity and maintainability. A bit of accounting can help us get the discussion started. Say we start off with a single document type that is in version `v1`. We now update the schema to `v2`, so the app will need an adapter to deal with the older `v1` documents. After the next update to `v3` the new app will now need two adapters: one to deal with `v2` documents and one to deal with `v1` documents that may also still be around. In general, every app that has ever existed in the system may have left documents in the corresponding old versions around. Since we can never be sure that there are no ancient schema versions around we will have to provide `n - 1` adapters for an app that uses data schema version `n`.
But there's more. Since clients can be offline or not get updated, older versions of clients need additional adapters to migrate documents up to their specific version and those add up to what we have to maintain. To round off this part of the analysis, let's just say that all app versions that have ever existed may still be used somewhere, and accordingly all document schema versions that have ever existed will need to be supported. If the current schema version number is `n` we would need to provide

<math xmlns="http://www.w3.org/1998/Math/MathML">
  <mo stretchy="false">(</mo>
  <mi>n</mi>
  <mo>&#x2212;<!-- − --></mo>
  <mn>1</mn>
  <mo stretchy="false">)</mo>
  <mo>+</mo>
  <mo stretchy="false">(</mo>
  <mi>n</mi>
  <mo>&#x2212;<!-- − --></mo>
  <mn>2</mn>
  <mo stretchy="false">)</mo>
  <mo>+</mo>
  <mo>.</mo>
  <mo>.</mo>
  <mo>.</mo>
  <mo>+</mo>
  <mn>2</mn>
  <mo>+</mo>
  <mn>1</mn>
  <mo>=</mo>
  <munderover>
    <mo>&#x2211;<!-- ∑ --></mo>
    <mrow class="MJX-TeXAtom-ORD">
      <mi>i</mi>
      <mo>=</mo>
      <mn>1</mn>
    </mrow>
    <mrow class="MJX-TeXAtom-ORD">
      <mi>n</mi>
      <mo>&#x2212;<!-- − --></mo>
      <mn>1</mn>
    </mrow>
  </munderover>
  <mi>i</mi>
  <mo>=</mo>
  <mfrac>
    <mrow>
      <mi>n</mi>
      <mo stretchy="false">(</mo>
      <mi>n</mi>
      <mo>&#x2212;<!-- − --></mo>
      <mn>1</mn>
      <mo stretchy="false">)</mo>
    </mrow>
    <mn>2</mn>
  </mfrac>
</math>

adapters.

We're not done yet. So far, we have just looked at a single document type, but our schema can accommodate dozens of them. In our example we just had three types (`todo-item`, `status`, `settings`) but to be more general let's say we have `t` different document types. If we introduce a new version for every type with every update we need \(\frac{t n (n - 1)}{2}\) adapters in total or, amongst friends, $ \mathcal{O}(t n^2) $ adapters. This can quickly get out of hand and we have to look at optimizations and compromises.

#### No legacy-app dropping and no purging of old documents

TBD


#### Old apps have to force users to do updates

[note: implement this fro the start!]

This is probably the biggest concern with live migrations but also the hardest to work through so we will discuss this last. It means that an old app has to be shut down until it is updated. For a simple web app this may be as easy as requiring a page reload, for a desktop app it may require a restart, but for iOS or Android apps it may require users to visit some appstore and go through a whole procedure for upgrading an application. And what if product decides that updates should be paid for by device? Do we shut down all older apps once someone upgrades a single device?

These are serious concerns that arise because we are unable to meet two requirements at the same time:

1. Make sure newer documents do not end up in older applications' databases. If this is violated, the old app will not know how to handle the new documents and may crash.
2. Make sure documents are synchronized with all relevant associated documents. If this is violated, there may be inconsistent data. Imagine if we had just a status, but the associated todo item was missing, or if we had an address without a user, or a text message without a contact.

It would be possible to enforce the first point on its own. CouchDB provides a mechanism to synchronize only a filtered set of all data (that's called a [filtered replication](...)), and we could use this to exclude data that is too recent to get to clients that are too old. But this would conflict with the second point. If we update only the todo items but not the status then a filtered replication would sync the status documents but filter out the corresponding todo items. This way, a database could end up with status documents without a todo item. The only way to work around this problem is to increase the schema version of *every* document once a single document changes. In a larger setting with dozens or hundreds of document types in the database this would lead to a lot of duplication.

##### Global Changes feed
How can one implement continuous server-side migrations on CouchDB you ask? Here's a sketch: First there must be a way to look at incoming documents so they can be updated in the first place. CouchDB provides such an option through the `_changes`-endpoint that allows us to keep track of every event that happens in a database. It is not a bad idea to bundle the different `_changes`-endpoints of all user databases together into a **global changes feed**. A backend service could then listen to this feed to see if any outdated document comes in and perform transformations as needed. After the updated document has been saved, there is no need for the old version to be around, so it can safely be deleted.

