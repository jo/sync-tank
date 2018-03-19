---
layout: default
permalink: /chesterfield-migration/
---

# [Sync<br/>Tank](/) Chesterfield Migration

## An eager server-side multi-version distributed migration strategy
{:.no_toc}

## Table of contents
{:.no_toc}

1. this unordered seed list will be replaced by toc as unordered list
{:toc}

## Chesterfield Migration

We have come far. We now share a common vocabulary and a systematic understanding of distributed migration strategies, we have seen when they are necessary and when you might get away without them. In this section we are going to take a good look at one particular strategy that can work well in practice. We will not shy away from addressing a range of problems and difficulties that emerge when this approach is implemented on top of CouchDB and we will propose a set of solutions for them, so be prepared for a more detailed technical discussion.

The strategy we are going to present is not the only reasonable choice as should be clear from the previous discussion. Still we believe that among the options we discussed it allows for a clean and maintainable implementation given that we want to support the complex scenario we have built up in the previous sections. To recap: we demand that our system support offline-capable applications and clients on multiple platforms that require different schema versions of application data, all the while providing full backwards-compatibility for older apps or services and enabling agile development.

<figure>
  <img src="/chesterfield-migration/images/chesterfield.jpg" alt="The authors: Matthias and Johannes" />
  <figcaption>
    <b>Chesterfield.</b>
    <span>
      A type of luxurious couch. And an appropriate name for a migration strategy that has you covered in all kinds of circumstances.
    </span>
  </figcaption>
</figure>

In order to have a memorable reference, we are going to brand our strategy as *chesterfield migration*. A chesterfield is a type of luxurious couch, and we feel this is an appropriate term as we are going to describe an approach that meets all kinds of differents requirements comfortably - and is based upon CouchDB.

### We are not alone

The solution we discuss here does not come out of thin air. It has emerged from a longer discussion with contributers from different institutions and the offline-first community. To give credit where credit is due, we would like to begin with a very brief recap of the background of our approach.

During his time with eHealth Africa Johannes began to think about the problem of distributed migrations together with Jan Lehnardt. When he later joined immmr, the company was still on its way to developing a market-ready version of its first product. At this time, there were a lot of concerns about the viability of schema migrations. The only real alternative to migrations - getting everything right from the start - has some problems of its own so Johannes and Ben Kampmann generated several ideas for migration strategies, from which the approach we are going to present here emerged as a final result.

<figure>
  <img src="/chesterfield-migration/images/offline-camp-migration-session.jpg" alt="Migration session at Offline Camp 2017 Berlin" />
  <figcaption>Foto by Gregor Martinus: Migration session at Offline Camp 2017 Berlin</figcaption>
</figure>

The [offline camp berlin 2017](http://offlinefirst.org/camp/berlin/) provided an excellent opportunity to discuss the strategy with members of the offline-first community and we gratefully appreciate the thoughtful comments from Gregor Martinus, Bradley Holt, Martin Stadler (a former immmr-colleage!) and others. These discussions gave us a lot more confidence that we have found a robust approach that can persist through a number of challenging edge-cases.

### An eager server-side multi-version migration strategy

The chesterfield migration is an *eager server-side multi-version migration*. Our implementation of this features, in broad strokes, a micro-service listening to CouchDB's `_changes`-endpoint for document updates and activating different *transformers* on demand which perform the actual document migration, all of which is happening on the server-side database with changes being replicated to client-databases afterwards. [Figure 3](#figure-3) illustrates the idea: when necessary, transformers create multiple versions of documents so that a single shared database can support multiple app versions.

<figure class="diagram" id="figure-3">
  <img src="/chesterfield-migration/images/per-version-docs.svg" alt="Schematic view of chesterfield migration" />
  <figcaption>
    <b>Figure 3: Chesterfield migration.</b>
    <span>
      Documents exist in multiple versions in order to support multiple application versions through a single database. Transformers are the engines that perform up and down migrations.
    </span>
  </figcaption>
</figure>

This setup is rather complex, involving a number of new ideas and several moving pieces. It will therefore benefit our discussion to take a moment and illustrate the situation using our working example. In the previous sections we have developed a simple todo app that has passed through a few iterations by now. The schema has evolved to accomodate an item's `isImportant` and `status` values. Because our current problem is quite involved, we would like to introduce yet another feature that comes with yet another breaking schema change, so that there is a bit more material to work with.

```cucumber
As an app user
I want to group todo items
so that I can stay organized even when there are a lot of things to do.
```

To keep things simple, we will require each todo item to belong to exactly one group. This association can be established via an additional `group`-attribute that stores the respective group name for each todo item. If no particular group is chosen by the user, items should be assigned to a group named "default". You could argue that this didn't have to be a *breaking* schema change, that it would be possible for apps to handle older items without `group`-attributes as though they were *default* items. But let's assume for the sake of better illustration that in our case this is not good enough for app developers. They *need* every todo item to state its group. And as we argued that app expectations define a data schema to begin with, so they also define what counts as a breaking change. To make a long story short: we need to *require* a `group`-attribute for each todo item, and we need to go over all existing items and add them to the default group. So we indeed need to run another migration.

For completion, here's a valid todo item document according to version three that stores its associated group. Notice how we keep track of the document version in the `_id`, an addition we will come back to shortly.

```json
{
  "_id": "todo-item:c2b2dc0a123d27752bc08ad39d000bbe:v:3",
  "schema": "todo-item",
  "version": 3,
  "title": "Add all todo items to the default group.",
  "group": "default",
  "createdAt": "2018-31-01T14:49:12.071Z"
}
```

The grouping feature has forced us to introduce another major schema version. We are now dealing with three schema versions, and accordingly we need to maintain all three of them in parallel, in order to support all the apps out there that have never been updated. The following listings give a quick overview over the three schema versions using the manifest-notation introduced in the schema-section above.

The first major version started out with the bare todo items and introduced `settings`-documents and the `isImortant`-flag as two features:

```json
{
  "name": "todo-app",
  "version": "1.2.0",
  "dependencies": {
    "settings": "^1.0.0",
    "todo-item": "^1.1.0"
  }
}
```

The second major version introduced a separate `status`-document while removing the `isDone`-property from todo items:

```json
{
  "name": "todo-app",
  "version": "2.0.0",
  "dependencies": {
    "settings": "^1.0.0",
    "todo-item": "^2.0.0",
    "todo-item-status": "^1.0.0"
  }
}
```

The third major version that was just introduced requires todo items to have a `group`-property, hence the todo item schema had to be updated once more:

```json
{
  "name": "todo-app",
  "version": "3.0.0",
  "dependencies": {
    "settings": "^1.0.0",
    "todo-item": "^3.0.0",
    "todo-item-status": "^1.0.0"
  }
}
```

These are the three major schema versions that we have to maintain in parallel. In order to keep all versions up to date when one piece of data changes it will be necessary to update existing documents in multiple versions. For instance, if a user creates a todo item with their very old app that still runs version one, we need to migrate documents up so that the new todo is not lost on devices that run newer versions. Similarly, we need to migrate changes made with newer apps down to support older apps. Apart from the actual migration logic this setup requires some additional infrastructure to work well. First, we will need to distribute documents with multiple versions throughout the system, and then we will need to manage documents with multiple versions from inside the applications. Let's take a closer look at these two tasks.

#### Replication channels

Chesterfield is a *server-side* migration. This means that apps will replicate documents from their local databases to the server-side database where *transformers* will produce all relevant versions of the documents that will then be synchronized across the system. If we dig a little further though, we will find that not *all* versions will have to be propagated to *all* clients. For instance, if a new client produces a todo item according to version three, and if the server migrates this document down to versions one and two, the newer client would not be interested in receiving the older documents. We can save a lot of traffic and storage if we can prevent newer clients from receiving older documents.

Do we also need to prevent clients from receiving *newer* document versions than they can currently handle? The answer depends on which side we favor in a trade-off between replication traffic and ease of application update. On the one hand, preventing newer documents from being replicated to clients that are not yet ready results in lower traffic. On the other hand, the newer documents will have to be replicated anyway once the application is updated and is expecting newer schema versions.

And there is more: if we allow newer documents to reach clients that are not yet ready we can support what we would like to call *seamless migrations*, migrations without downtime. This is possible because we could migrate documents up to a new version, distribute them across the system, and only once the clients had enough time to receive the new data would we update them. Once the apps get updated, they find all the new data they need is already in their local database!

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

There are a few noteworthy aspects about this selector. We said it's supposed to be used by clients that work with schema version two. Recall from the example manifests that schema version two requires todo items of version two. But since we want apps to receive all newer documents as well we ask to exclude lower versions from the replication. Similarly, this selector will also replicate all documents that are not todo item documents. This includes status and settings documents of version one and higher as well as all other types that may be added in the future. This way, clients can stay open to new schema versions and be prepared for the next seamless migration.

As a last point we'd like to reiterate that replications take time and may lead to temporarily incomplete data. For instance, the todo item may have already been replicated while the corresponding status document is still missing. But this is a general learning about CouchDB: clients have to be able to deal with this kind of incomplete data anyway. If you really need pieces of data to be present together consider keeping them together in one document.

#### Multiple schema versions in a single database

After successful replication clients will have all necessary data in their local databases, although potentially in multiple versions. This raises the question of how parallel schema versions can be managed within a single database.

An obvious first requirement is that every document's `_id` now has to reflect the version number. This was already addressed when we introduced our taxonomy above. *Single-version* and *multi-db* migrations do not have to modify document identifiers - they can simply reuse them across versions. *Multi-version* migrations, however, cannot afford this convenience. One option for reflecting the version in the `_id` is to simply add it at the end. This would for instance create todo items with `_id`s like `todo-item:ce2f71ee7db9d6decfe459ca9d000df5:v:2`. A transformator performing an up-migration of this document would accordingly have to increase the version number to 3 without modifying the hash so the document could still be identified.

Be careful, though, not to rely on document versions when it comes to imlementing associations. If we associate, say, an address with a user through a `userId`-attribute on the address, this should not specify the version of the user document. Otherwise migrations could destroy existing associations.

Apart from reflecting the schema version in the document identifier we have made sure from early on to keep `schema` and `version`-attributes in each document. This will pay off now as clients need to retrieve exactly the version of a piece of data that they can currently work with. Traditionally most data was retrieved from CouchDB via *views* but once again Mango selectors are a recent, very performant alternative. By way of an example consider a client working with data schema version two that wants to display a list of todos. The same todo may exist in multiple parallel versions in the database, but the client can retrieve exactly the version it knows how to handle using the following selector.

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

These three guiding principles - reflecting the document version in the identifier, establishing associations through raw ids, and querying documents by relevant `schema` and `version` - allow  clients to handle multiple parallel schema versions in a single database. All this is possible without having to change any URLs when communication with the server-side database.

Before moving on to see how the transformer engine works under the hood we would like to address a concern you may have regarding the size of local databases. Memory is still a somewhat scarce resource especially for mobile clients. Would the approach we have sketched out here not bloat client databases unnecessarily by keeping legacy schema versions around when they are in fact no longer needed? As user data builds up, this may indeed become a problem at some point. If that's the case, don't despair, as there are ways to perform some data compaction. We suggest creating an additional local database on the client and running a filtered replication that replicates only relevant document versions. After that, switch over to the new database and discard the old one alongside all legacy schema versions.

### Transformer modules

We are finally ready to take a close look at transformer modules, the main workhorse of our migration strategy. Transformer modules, or transformers for short, live on the server, they are responsible for noticing when a document has to be migrated and they create and persist new document versions. These are three different tasks - listening, creating, and persisting - and our transformers will be composed of three corresponding components. To be more precise, each transformer module will consist of a *registry*, a *migration logic unit*, and an *update handler*. Let's introduce each of those in turn and see what considerations have led us to design them the way we did.

#### The registry

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

#### The migration logic unit

```
- schema knowledge: how to get it, what is needed
- splitting up documents
- merging documents
  - needs retrieval
    -> more views?
- caveat: the concept of document-level transactions which emphasizes the need to keep information that *must* be available together in a single document and otherwise embrace the idea that updates might only happen incrementally and might even lead to temporarily inconsistent data if not treated carefully.
```


#### The update handler

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

