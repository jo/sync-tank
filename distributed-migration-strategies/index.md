---
title: Distributed Migration Strategies
layout: default
version: 1.0.0-rc1
publishedAt: Wed 21 Mar 18:42:35 CET 2018
# in vim `:r! date`
lastUpdatedAt: Thu 22 Mar 18:04:02 CET 2018
permalink: /distributed-migration-strategies/
---

# [Sync <br/>Tank](/) Distributed Migration Strategies
## Handling Schema Changes in CouchDB
{:.no_toc}

published {{ page.publishedAt | date: '%B %d, %Y' }}, last updated {{ page.lastUpdatedAt | date: '%B %d, %Y' }}

## Table of Contents
{:.no_toc}

1. this unordered seed list will be replaced by toc as unordered list
{:toc}

## Introduction

> "Software development is change management" - Ashley Williams, [A Brief History of Modularity](https://www.youtube.com/watch?v=vypCsVm5z28) at JSConf EU 2017

Good software is constantly changing. Users will inevitably suggest improvements and make change requests. The only software that does not change over time is software that nobody uses. That's the first part of our problem. The second part is the sort of software we would like to be writing: offline-capable, decentralized and scalable systems that can support and synchronize a whole range of different clients.

There are separate solutions and best practices for both of these challenges. Agile methodologies integrate readiness for change into the very process of developing software, allowing us to have working software in front of customers early on and adapting it based on feedback. There also exist tools and technologies that support offline-capable multi-client applications. But as it turns out it is not so easy to bring those requirements together. The cause of all the trouble is one of those parts of the system that is usually friendly and not as attention-craving as shiny front-ends or massive costs of operating server clusters: the data schema. The data schema?

Here's the short version of the problem: if an application implements new features, it often requires new data or data to be in different formats. That's a schema migration. Now if you have multiple applications that share data across the system, even more so when some of those could be offline, it is impossible to update all of them at that same time. This leads to our initial conundrum:

* How can we update the data schema to support new applications when there might be old applications around that still rely on older versions of the schema?

At the time of this writing, performing schema migrations in distributed systems is largely unexplored terrain. In what follows, you will find a detailed exposition of our thoughts on this problem. We will outline various solutions from simple to more complex ones, and build up to a taxonomy of what we call *distributed migration strategies*.

To make the problem more manageable, and because our professional background naturally leads us to it, we will focus our discussion on *Apache CouchDB*, especially when it comes to providing examples for the topics we discuss. And while we're at it, let us add a note on our background. We, Johannes J. Schmidt and Matthias Dumke, are data architects at a company called [immmr](https://www.immmr.com/) that builds communication applications for a variety of platforms. Our main task is to manage data synchronization across a large, distributed system of native clients, web applications, desktop apps, and backend services. In this context, CouchDB is our main tool when it comes to providing stable and scalable replication of user data and allowing the company to stay agile.

In this article, we will begin by going through some basic concepts before we will illustrate the problem of schema migrations, starting from simple scenarios and integrating more complex requirements as we go along.


## Preliminaries: CouchDB, Schemas, and Migrations

Going through a technical discussion will be more fruitful if everybody is on the same page from the beginning. This is why in this part we would like to address a handful of general topics including a refresher on CouchDB, our understanding of schemas, migrations, and distributed systems, as well as best practices for working with data schemas in the wild. If you are already familiar with these topics, feel free to skim or skip this section. We will begin our discussion of migration strategies in the next part.

### CouchDB - A Database that Syncs

As we just mentioned, and as the title of this article already suggests, we will narrow down the focus of our discussion to software systems backed by CouchDB. In case you didn't know,

> "Apache CouchDB™ lets you access your data where you need it by defining the Couch Replication Protocol that is implemented by a variety of projects and products that span every imaginable computing environment from globally distributed server-clusters, over mobile phones to web browsers."
>
> From the official [Apache CouchDB Website](https://couchdb.apache.org/)

We choose to focus on CouchDB because first of all, there is basically no production-ready alternative for a cross-platform data storage that provides synchronization capabilities and is also open source. Check out this [comparison chart](http://offlinefirst.org/sync/) to get an overview of offline capable storage options. And secondly, and pragmatically, we already know a lot about working with CouchDB, so this seems like a reasonable place for us to begin the discussion.

CouchDB is a scalable document-oriented store that allows efficient MapReduce data queries via indexing. It is a mature open source project under the Apache foundation that supports clients on all major platforms including [web browsers](https://pouchdb.com/), [Android](https://github.com/cloudant/sync-android) and [iOS](https://github.com/cloudant/CDTDatastore) devices. One of its main features is reliable synchronization of data between different installations via multi-master replication. We do not assume in-depth familiarity with CouchDB though some expose will be helpful. In the following, we will quickly go over the most important concepts that are relevant in the context of schema migrations. If you need a more in-depth introduction to the system, [CouchDB. The definitive Guide](http://guide.couchdb.org/) is a good place to start.

To begin with, let's get some technicalities out of the way: CouchDB provides an HTTP API for all database interactions like storing or retrieving information and managing databases. A single database within the system is a document that can be created via a simple HTTP request like so: `PUT $COUCH/db-name`. This means that databases are very lightweight and can be created and deleted with ease.

Data is transmitted and stored in the form of JSON-documents that can have any valid format you want them to have, plus some attributes such as `_id` and others starting with an underscore that have special meaning. Here's an example document to make this more concrete:

```json
{
  "_id": "todo-item:02cda16f19ed5fe4364f4e6ac400059b",
  "title": "Moonwalk the dog"
}
```

This is a JSON-document storing information about a todo item. We will see more of these documents when we start developing an example application in the next part. But for now, we want to focus just on the `_id` attribute, the document identifier.

CouchDB uses `_id` to uniquely identify documents. If there is any information you need to be unique throughout the database, put it into the `_id`. If you don't set one yourself, CouchDB will create a UUID for you and set it for you, but we recommend taking the opportunity and storing more meaningful information here, in other words: making the `_id` semantic! In the example, we store the document type along with a UUID. This allows us to quickly identify the kind of document we are dealing with if this id is returned e.g. from a query. It also allows us to extract documents by type via the handy [`_all_docs`](http://docs.couchdb.org/en/latest/api/database/bulk-api.html#db-all-docs) query. Finding a good strategy for creating `_id`s is one of the challenges when it comes to data design. We will touch upon this topic a couple of times further down the road.

As was briefly mentioned above, CouchDB makes it possible to retrieve documents by implementing the powerful [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) pattern. In particular, it allows us to create so-called 'views' that define which data should be returned when they are queried. CouchDB will then build indices to find the requested data quickly and efficiently. As of version 2.0.0 CouchDB also allows queries via [Mango selectors](http://docs.couchdb.org/en/latest/api/database/find.html) that are based on MapReduce and are very performant yet flexible. The ability to query efficiently is of central importance when it comes to data design but there is no need to go into further depth at this point because the discussion of migrations will not require that.

There is one more topic we have to address here if we are going to talk about migrations further down the road, and that is *conflicts*. Conflicts can arise when data is being synchronized between multiple instances of the same database. To get a feeling for when this may happen, imagine the following 'split brain' scenario: Our friend Maja has CouchDB running on two devices and she opens the database where she stores her favorite things. Both instances are in the same state initially, and both contain the same document:

```json
{
  "_id": "favorite-programming-language",
  "_rev": "1-acabacabacabacabacabacabacabacab",
  "name": "Elixir"
}
```

Notice the `_rev`-attribute in this document. It's another one of those that have special meaning to CouchDB. Its value consists of two parts, the 'revision number' and the 'revision id', and the format is always `[revision-number]-[revision-id]`. This attribute is used to keep track of how a document changes over time. Every time a change happens, the revision number is incremented and a new revision id is assigned. Let's see this in the example.

If there was a network connection between Maja's devices, any changes she'd make to this document on one of them would be replicated to the other so that her data would be synchronized. However, let's assume there is currently no network connection. Now Maja goes and changes her favorite language on both devices to something different (in order to break the system, of course, not because her preferences are changing so rapidly). So here is the document as it is stored after the changes on her first and second device respectively:

```json
{
  "_id": "favorite-programming-language",
  "_rev": "2-bdec0dcd2c295a50d54556ebdca300cd",
  "name": "Lua"
}
```

```json
{
  "_id": "favorite-programming-language",
  "_rev": "2-4ab8a90d4c7c68221719559660a477ff",
  "name": "Lisp"
}
```

In both cases, the revision number is increased to two and the name attribute has been changed, of course. Now if both devices are connected CouchDB will try to bring all data to the same state, but it is inherently unclear which one of the changes should determine the final version of the language document. Even if it was feasible to compare document write times across different systems (which is often very hard to accomplish) and then choose a final document on a last-write-wins basis, it is not clear that this manner of handling conflicts is always the preferred way. This is why CouchDB will by default pick some version as winning - deterministically based on the revision id and number - and will reference conflicting versions in the document. This way, no information is lost and conflicts can be resolved more elaborately if necessary. For completeness, here is the final document that CouchDB automatically creates after default handling of the conflict.

```json
{
  "_id": "favorite-programming-language",
  "_rev": "2-bdec0dcd2c295a50d54556ebdca300cd",
  "name": "Lua",

  "_conflicts": ["2-4ab8a90d4c7c68221719559660a477ff"]
}
```

The next time you retrieve this document, you can instruct CouchDB to also include any conflicting revisions and it will include the `_conflicts`-attribute. Based on this it is now possible to implement mechanisms for resolving conflicts in ways that fit your needs best. This is great: CouchDB will not lose any information, it provides reasonable defaults, and it still gives you full flexibility over how to resolve the conflicts you encounter.

For the time being, these notes will suffice as a review of central CouchDB concepts. We will touch upon many of the points throughout the rest of this article. Next up is an explanation of what we have in mind when we talk about schemas, migrations, and distributed systems.


### Basic Concepts: Schemas, Migrations, and Distributed Systems

If you want to store and retrieve data in an automated and efficient way, it is important to have some knowledge about the format or structure of this data. This is meta-information specifying things like data attributes and the data types to be used to store them. We would like to think of the **data schema** as all relevant structure-information about the different pieces of data to be stored by an application.

On this general account, a schema formalizes the structure of your data. Many database systems actually require in advance an explicit account of what the data to be stored is going to look like. PostgreSQL, for example, allows storing relational data where entries have to adhere to one of the [available data formats](https://www.postgresql.org/docs/8.4/static/datatype.html). These formats will even be validated on store-time.

Document databases like CouchDB are sometimes referred to as *schemaless*. This only means that the database will not force you to make your schema explicit before storing data, it does not mean that your data will not have a schema. In the end, a schema is not defined by the database anyways, but by an *application* expecting data to have a certain format. If you don't make your schema explicit, it will simply be *implicit*.

> "Usually when you're talking to a database you want to get some specific pieces of data out of it: I'd like the price, I'd like the quantity, I'd like the customer. As soon as you are doing that what you are doing is setting up an implicit schema."
>
> Martin Fowler, [Introduction to NoSQL](https://www.youtube.com/watch?v=qI_g07C_Q5I)

Thinking in terms of implicit schemas as defined by the application's expectations may require a change of perspective, but it will benefit us in the long run. Moreover, in an upcoming section, we will recommend that schemas be made explicit regardless as this will allow us to be more precise about what some piece of data looks like and to keep track of how it changes over time.

Once the application evolves and starts to make new assumptions about the data, the flexibility of the (implicit) schema becomes an issue. With a rather lenient document database like CouchDB it may be possible to simply store some additional information in documents at first, but after a while, some documents will become outdated and unable to support the functionality new app versions require. At this point, we will be forced to transform existing documents into a new form (whether we overwrite existing data, transform it on-the-fly, or find some other way to perform those updates remains a question to be discussed). Changing existing data to adhere to new requirements is called a **data migration**.

The last concept to address are **distributed systems**. According to the Wikipedia,

> "A distributed system is a model in which components located on networked computers communicate and coordinate their actions by passing messages."
>
> Wikipedia, [Distributed computing](https://en.wikipedia.org/wiki/Distributed_computing)

To apply this broad definition to the CouchDB environment, we can interpret database documents as messages, data transfer via replication as message passing, and the various apps on phones and in browsers and the backend services as the components located on networked computers. This can mean that data will have to be stored not only on a server but also on user-devices, which in turn must be enabled to synchronize data updates with the rest of the system so all parts can operate on a shared informational basis.

In this distributed scenario, different applications can make very different use of the same data. For instance, an Android app may use data to display a list of todos to the user while a backend service may be interested in the metadata to build usage profiles. We, therefore, have software that accesses, processes, and stores data in very different locations and according to their very different requirements. For the whole system to be intact it is mandatory to not change the structure of the data in any unforeseeable way. The different parts of the system, and consequently the different teams working on the different parts of the system, are bound by an (implicit) contract, by the (implicit) data schema they all have to respect. In the scenario we are going to build up here, we may even go one step further and say that *the data schema is the effective database API* of the distributed system because it ultimately defines the way in which data can be accessed. From this perspective, a schema-change implies an API change for any part of the system that is dependent on the data. This is why it is of such importance to have a strategy for dealing with schema changes.

When migrations become necessary in distributed systems, we run into the complex issues this article is about.


### Make Your Schema Explicit!

We have already mentioned that even if people may call your database *schemaless* this does not mean you can actually work without a data schema. Granted, you may be able to change your mind at any time and store differently structured information and the database will be fine with that. This is why schemaless databases have a reputation for facilitating flexible development. But at the very least the data schema will be *implicit* as your applications have certain expectations about what kind information there is in the database. We have found that it is very helpful to be more explicit about the schema. Allow us to argue for that:

1. A data schema that is made explicit makes **communication** about data a lot easier. This is true across time (when you come back to your project three months later) and across teams (when the web team is supposed to add the same functionality to their apps as the Android team).
2. An explicit schema also allows for format checking and **schema validation**, which can expose bugs in applications earlier, and in general lead to more reliable and consistent data. More reliable and consistent data makes application development not only easier but practically feasible in the first place.
3. Finally, an explicit schema can be **versioned** which makes it possible to track how the format changes over time and even to revert to earlier states.

If this has been convincing to you a plausible follow-up question is how to actually make a data schema explicit. There are a number of tools around to help with that. As this article focuses on CouchDB let us first mention that the CouchDB team [has plans](http://markmail.org/message/54puucfxsjob57zw) to introduce server-side validations via a *Mango*-like syntax query language. This can be used to enforce requirements on the structure of data before storing it. You may also have come across [JSON schema](http://json-schema.org/), a very flexible schema specification vocabulary. In case you are not familiar with this, here's a simplified example of how we could make the schema of a hypothetical todo item document explicit using JSON schema:

```json
{
  "id": "todo-item",
  "version": "1.0.0",

  "properties": {
    "_id": {
      "pattern": "^todo-item:[a-f0-9]{32}$",
      "type": "string"
    },

    "schema": {
      "enum": ["todo-item"]
    },

    "version": {
      "enum": [1]
    },

    "title": {
      "maxLength": 64,
      "type": "string"
    },

    "isDone": {
      "type": "boolean"
    },

    "createdAt": {
      "type": "string",
      "format": "datetime"
    }
  },

  "required": [
    "_id",
    "schema",
    "version",
    "title",
    "isDone",
    "createdAt"
  ]
}
```

This specification formulates some expectations and requirements any valid todo item document would have to fulfill. For instance, it requires each such document to set a `title` attribute which must be a string of some maximum length. It is also possible to specify more complex formatting rules the pattern of the `_id` property exemplifies. CouchDB integrates nicely with JSON schema: the above specification could be used to validate documents via the `validate_doc_update` [function](http://docs.couchdb.org/en/2.1.1/ddocs/ddocs.html#validate-document-update-functions). To round things off, here is what a corresponding valid document might look like:

```json
{
  "_id": "todo-item:02cda16f19ed5fe4364f4e6ac400059b",
  "schema": "todo-item",
  "version": 1,
  "title": "Feed the trolls.",
  "isDone": false,
  "createdAt": "2017-01-20T17:00:00.000Z"
}
```

The example features `schema` and `version` attributes on the document with values of `todo-item` and `1` respectively. This will be most valuable information once we have to implement schema migrations. We also recommend maintaining a schema version for each document type *separately*. The version is then tightly coupled with the data format while leaving enough room to change document versions independently of each other. For instance, we could upgrade `address` documents from version `1` to `2` while the related user data could still be stored in `user` documents of version `1`. This prevents us from having to increase schema versions unnecessarily.

Being explicit about schemas is also the first step to *versioning* them. As a final practical recommendation, we suggest to use [semantic versioning](http://semver.org/), properly interpreted to work well with data schemas, in order to keep track of how a schema evolves. We use semantic versioning to determine the version of the *database schema*, which is the collection of all *individual document schemas*. This way, it is easy to understand the type and severity of schema changes:

* **Breaking changes** happen every time a schema update breaks some part of the system.
* **Features** are all enhancements that do not stop existing applications from working or render existing documents unusable.
* **Fixes** are small improvements like correcting typos in schema descriptions.

Determining which schema changes count as features or as breaking changes is not as simple as it may seem at first glance. There is one general rule that is straightforward to establish: making schema specifications stronger *always* introduces a breaking change. The opposite is not always true, though. A weaker specification may introduce a breaking change as well - if we want to support older versions of an application. If we weaken a specification it is possible that older apps cannot work with newer documents because they will miss a formerly required attribute or don't know what to do with this new Enum value we allowed. In the end, detecting breaking changes boils down to asking the simple question: which applications do I want to support and how does the change I'm about to introduce affect them?

In the above example, the schema specification for the document has its own version, `1.0.0`, which will be updated as the schema evolves. With versioning in place on the document level, it is only a small step to creating a schema manifest that bundles and specifies the document versions that make up a schema version. To be concrete, we could use a good old `package.json` to specify that user, address, and todo item documents are the constitutive parts of the database schema for an exemplary todo application.

```json
{
  "name": "todo-app-schema",
  "version": "2.1.0",
  "dependencies": {
    "address": "^2.0.0",
    "todo-item": "^1.1.0",
    "user": "^1.3.1"
  }
}
```

To sum up this discussion, introducing format-requirements to schemaless databases opens up a space between purely implicit schemas and rigidly explicit ones, providing greater flexibility when it comes to specifying and changing schema definitions. This allows us to pick the sweet spot between data consistency and flexibility that we see fit for our projects. Storing the schema version of each document in the document itself paves the way for implementing schema migrations, which we will now turn our attention to.


## If It Ain't Broke, Don't Fix It

There are times when updates to a data schema do not endanger the functionality of a system. In these scenarios, migrations are either straightforward or not even necessary in the first place. We would like to start by looking at some such cases because if you can get away without the multi-client multi-version migration hassle you can save yourself a lot of work. In particular, we will start by looking at unobtrusive additions to the data schema as well as traditional server-side migrations.

### Setting the Stage: A Toy Problem

While this article is not supposed to be a tutorial, it will be instructive to have a working example at hands to illustrate some points. Our example of choice is the 'Hello World' of software applications, the todo app. A simple todo app does not look like the most daunting challenge to face from a data architecture perspective: todos can have titles and texts, maybe a creation date and a 'done' flag. None of this would make you sit down and write a full article about different strategies for how to accommodate this information in your database. But things get a lot more interesting once we agree to meet a few additional requirements:

1. Users should be able to edit their todos even when their internet connection is unstable or down, and the application should synchronize all changes once it reconnects. In other words: the app should be **offline-capable** (what exactly this means will become clearer as we go along).
2. The application development should be **agile**, meaning that we want to support frequent changes to all aspects of the system, including the data schema.
3. Users should be able to use the todo app on **multiple clients**, for instance as a web service and as an iPhone app, with potentially different versions of the software running on different clients.

These requirements will eventually lead us to think about distributed migrations. Of course, this is all way ahead. For now we just want you to know that a tiny todo app can cause a whole lot of trouble further down the road. But this section is about starting simple, so let's start with a simple setup for a simple todo app and take on new challenges as we go along.

In the simplest of todo app scenarios, we want to enable a user to manage a few, or maybe a few thousand todo items from the comfort of their web browser. The basic building blocks to set up such a service are a web app to provide a nice interface, a server to deliver that application, a central database that stores all those todos, and a bit of infrastructure to glue the pieces together. We have decided to use CouchDB as our database and as we mentioned above we chose to set up the system such that every user gets their very own database. This approach ties in well with the fact that CouchDB permissions-management is modeled on a per-database basis. This practice, known as *db-per-user*, is very common in the CouchDB world, though it is [not beyond dispute](https://medium.com/ibm-watson-data-lab/cloudant-best-and-worst-practices-7ee2040da1b). It entails that there is no way for one user to get direct access to anybody else's data because they are stored in different databases. Everybody manages their own todos in their own database. And yes: once our product goes viral and there are two million users there will be two million databases. Well, Ops problem. In fact, Ops will be happy to find that CouchDB comes with clustering abilities so scaling up is not a terrifying prospect - unlike when you run out of space with your relational database (though, to be fair, management may not be as happy, because operating a CouchDB cluster at scale may become a somewhat costly affair).

For now, this piecemeal approach of worrying about one user and one database at a time simplifies our problem. To complete the first step and bring version one of our todo app to the market, all there is to do as far as the schema is concerned is to decide how a single todo item is supposed to look. And since we wanted to start simple, and since the *sine qua non* of a todo item is basically just a title and a flag, here's an example of the first, launch-ready version of a valid todo item document to be stored in a user's CouchDB that also adheres to the schema we specified in the last part:

```json
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "schema": "todo-item",
  "version": 1,
  "title": "reimplement my pet project in Rust",
  "isDone": false,
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

### The World is Changing: New Requirements

> "Document databases are really cool… until you have to make a breaking change to the schema. Then it feels like “good luck with all that!” :D"
>
> Ben Nadel [on Twitter](https://twitter.com/BenNadel/status/918604059304779776)

The first weeks have passed, marketing has done a great job and our app is quite popular, especially with young professional single parents in urban areas. Feature requests are coming in and a decision is made to enhance the product. If we are lucky, we get away with some amendments to the schema we already have, but let's go step by step.

#### Version One: Emphasis and Theming

As a first new requirement, users would like to put special emphasis on some of their todos.

```cucumber
As a web app user
I want to mark a todo as important
so that I can find it easier.
```

The data schema will need some enhancements in order to store that new information. In particular, we decide to add an `isImportant` flag to each todo item. This change is rather unobtrusive because it leaves already existing todo items intact: since existing documents will not have the `isImportant` attribute, we can simply treat them as not important by default. All we have to do is make sure that new version of the app will be able to handle missing `isImportant` flags.

This change was not too hard to implement. A second request that many users have made is the ability to change the color theme of their app:

```cucumber
As a web app user
I want to choose between different color themes
so that I can express myself by personalizing my tools.
```

Obviously, this feature has no direct implications for the structure of todo items. We decide to introduce a new document type, a `settings` document, that will store the color information and perhaps other general preferences that will come up in the future. Since we only want a single settings document per database, there is no need to pick some unwieldy uuid as `_id` property. We can go for something simple instead, as the following listing of an exemplary `settings` document illustrates.

```json
{
  "_id": "settings",
  "schema": "settings",
  "version": 1,
  "color": "#e20074"
}
```

As with the introduction of an `isImportant` property, the new settings document is rather unobtrusive. Since it has just been introduced, there are no old versions around that could cause trouble for the new version of the app. And the previous version of the app would not know what to do with those documents anyways and could simply ignore them. As a general rule, creating documents with unknown types should never affect applications. For CouchDB, which relies heavily on views to retrieve data, this behavior can be achieved by *whitelisting* documents at the access level. In other words: when building up a view, confirm for each document that its type is known, and ignore it otherwise. This way, new document types can be added without affecting existing behavior.

#### Version Two: Progress Tracking and Groups

So far we have amended the todo item schema and introduced a new document type. Both operations are non-breaking feature changes to our data schema according to semantic versioning. But now let's look at yet another feature request that will have a deeper impact on our data format:

```cucumber
As a web app user
I want to assign one of many states (`active`, `blocked`, `done`, ...) to a todo item
so that I can have fine-grained control over its progress.
```

We already have the boolean `isDone` property that keeps track of an item's progress. But this progress information will now be stored in a different property, we'll call it `status`, that uses Strings to store the current state. So we are essentially replacing the old `isDone` with the new `status` property. Now older, already existing documents will *not* have a `status`, of course, and the new version of the app is built to *only* work with `status` and not with `isDone` anymore. This is a problem indeed. It looks like we have to find some way to update the old documents, to turn a todo document like the one above into a new form like the following:

```json
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "schema": "todo-item",
  "version": 2,
  "title": "reimplement my pet project in Rust",
  "status": "started",
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

As we are still wondering about how to solve the problem of transforming the old documents in the user databases into new ones, product knocks on our door with the latest new requirement.

```cucumber
As a web app user
I want to assign each todo item to a group
so that I can isolate tasks that belong together in order to be more focused.
```

Every todo item needs a group. So we could require a `groupName` property for todo items. However, we anticipate that people may want to add more information, for instance a group description, later on. This is why we try to be future-proof by embedding a `group` document into each todo item. This freedom to nest one document within another is one of the perks of working with a document oriented database, so why not make use of it.

Let's say developers have convinced us to *require*, not just *allow*, the additional group information. They argue it makes app development easier if every todo item carries its group information around. Requiring more information means we have to introduce another breaking change because old documents will not be assigned to any group. Luckily we have not released the new schema version yet, so we can bundle the two breaking changes into one release. If we assign it to a default group, the todo item from above now looks like this:

```json
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "schema": "todo-item",
  "version": 2,
  "title": "reimplement my pet project in Rust",
  "status": "started",
  "group": {
    "name": "default"
  },
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

#### Version Three: Group Avatars

We thought we would be future-proof, at least for a while, by using an embedded group document. But as soon as the new release is on the market, people are starting to ask for group avatars.

```cucumber
As a web app user
I want to add an avatar to a group
so that I can group todo items by emotional content.
```

This is a problem for our schema, of course. Embedding group information into each todo item document is doable, but embedding an image would require us to repeatedly store the same image information over and over again with every new todo item. This is not acceptable. Instead, we have to move the group information out into dedicated group documents that can be shared among many todo items. This will lead us to our next breaking change.

There is a straightforward and time-proven way to solve a grouping task in this way: *one-to-many associations*. We can create a document for each group and then link todo items to groups by storing a group identifier along with each item. The identifier that establishes this link is sometimes referred to as a *foreign key*. Before looking at the schema implications of this approach, we would like to add a more general note on associations in document databases because we have seen how this topic can trip up people new to the technology.

*Associations* describe connections between entities: *one-to-one*, *one-to-many*, or *many-to-many*. In *relational database design*, associations are typically established via *foreign keys* as described above. This pattern is so common that people sometimes believe it actually defines what a relational database is in the first place. Then they are surprised to see it appear in other places as well: "How can there be foreign keys in CouchDB documents? I thought it's not a relational database!" But associations are not what makes relational databases relational – *relations* are. A [relation](https://en.wikipedia.org/wiki/Relation_(database)) is a set of tuples of values with predefined types. That's all. There are operations like *selections* and *joins* defined on these relations (cf. [relational algebra](https://en.wikipedia.org/wiki/Relational_algebra)) that enable the complex queries and powerful query optimizations we all know and love. Associations play an important role in those operations, but the general concept goes far beyond relational databases themselves. Linking entities by their identifier is a very versatile strategy of data design, as we shall see below. Long story short: *relations* are not *associations*.

To prepare the `todo-item` and `group` association we first need to define `group` documents. So far, groups need to have a name and an avatar, which can be added as a an *attachment* in CouchDB. Here's an exemplary `group` document for illustration purposes, so that we are all on the same page:

```json
{
  "_id": "group:e302f183a96194f7d19dce0eaf5e3cf8",
  "schema": "group",
  "version": 1,
  "name": "Things I also want to do",
  "_attachments": {
    "avatar":   {
      "content-type": "image/jpg",
      "data": "01010101000111101010110001010"
    }
  }
}
```

In order to associate a todo item with a group, we can use the group document's id, which can be found in its `_id`-attribute. This way, a corresponding todo item could look like this:

```json
{
  "_id": "todo-item:4d3956b34778d92c676e3a487dad73fe",
  "schema": "todo-item",
  "version": 3,
  "title": "change the world",
  "isImportant": "true",
  "status": "blocked",
  "goupId": "e302f183a96194f7d19dce0eaf5e3cf8",
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

The `groupId` attribute clearly establishes the association of the todo item with the group above. It is also easy to see how multiple todo items could belong to this group if each of them specified the same group id.

We now have a good idea about how to adapt the data schema in order to react to the latest feature requests. We have even found a way to make developers happy by enforcing their expectations through schema validations. At this point we would like to release a new version of the web app that allows users (1) to mark todos as important, (2) to change the color scheme of their apps, (3) to choose from a number of progress states and (4) to sort their todo items into groups.

But wait! We have not talked about the problem of how to update existing todo items. Increasing the schema version for new documents is all very well, but which steps can we take to make sure that old documents can adhere to the new version? In principle, we know what to do:

1. We have to take all the old documents, remove the `isDone` property and replace it with a sensible mapping to some `status` (like `isDone` being `false` means `status` is `active`),
2. And then we have to assign each existing document to some default group by setting the `groupId` to the default group's id.

But the question remains: how do we accomplish this in practice? How do we actually run such a data migration?

### Traditional Server-Side Migration

We are now at a point where a new version of an application may be confronted with older documents in the system that it does not know how to handle. But this prospect does not scare us because there is a common practice that is used in this scenario, and one that you probably know well already. We will call this the *traditional server-side migration*.

<figure class="diagram" id="figure-1">
  <img src="./images/transactional-migration.svg" alt="Schematic view of traditional server-side migration" />
  <figcaption>
    <b>Figure 1: Traditional Server-Side Migration.</b>
    <span>
      Introducing a new version of an application may require us to update existing documents (white) to a new version (green). This switch happens at a single point in time. Some documents (two-colored example) can be handled by both app versions and do not need to be transformed at all.
    </span>
  </figcaption>
</figure>

Since we have stored all our data in one central database (because at this point we haven't required offline capabilities for our apps yet), it will be easy enough for us to access all existing todo items and update them to adhere to the new schema. Ruby on Rails's way of handling migrations provides a very straightforward example of this approach. In Rails, we would define a migration that formalized the schema change (prepare the database to store the new group document and group ids, update existing todo items with the id of some default group, and so on). We would then take the system down, run the migration (the famous `rails db:migrate`, formerly `rake db:migrate`) and hand out the updated application once the database was back up. If anything went wrong there would be a rollback because the migration would be wrapped in a transaction. During the process, we would, of course, incur some downtime, but on the plus side we'd always have consistent and up to date documents and everyone would get the latest version of our application.

This same strategy works with CouchDB as well. Let us roughly sketch out the main steps to go through when performing a **traditional server-side migration on CouchDB**. Here's our recipe:

1. Set up a new, empty cluster (in case you want to revert or abort the migration at some point).
2. Switch to maintenance mode.
3. Perform a replication of all data from the old to the new cluster.
4. Now run a migration script to update data on the new cluster.
5. Perform a check to verify that migration was successful.
6. In case of success: Switch to the new cluster and switch off maintenance mode. Done.
7. Else: 'Rollback' the migration by deleting the new cluster.

[Figure 1](#figure-1) illustrates the traditional migration strategy. It shows how both the application and the documents are updated together in a single step. White documents can be handled by the white version of the app while the green app needs green documents. A special case is the two-colored document. There might be documents that do not need to change during a migration and that can be handled by multiple versions of the client. Think of the settings document we have added previously. Even if we have to change the structure of todo items this does not mean we have to change how the settings document looks.

This migration procedure is very common for the type of monolithic centralized setup we have described so far. It does not come without some [problems of its own](https://about.futurelearn.com/blog/your-database-is-a-distributed-system) but overall it is a well established and reliable practice. Alas, this is not a viable solution anymore once we bring in some of the more challenging requirements we have omitted so far.


## Fail Fast, Fail Often

Failure tends to lead to success. When you fail with one approach, you can quickly develop new insights into a problem, and you can free your mind and pivot and try out alternatives. This process will often produce better results than sticking with one initial strategy could ever achieve. So we are going to fail in this part. In fact, we are going to fail a couple of times.

As a first step, we will bring back much of the complexity we have ignored so far. In this new context, we will be forced to look at more powerful migration strategies because our previous approaches are not viable anymore. In the spirit of this section, all the strategies discussed here will have serious drawbacks that should nevertheless prepare us well for a more systematic discussion that is coming up in the next section. Our primary goal, for now, is to develop a better understanding of the different types of problems that can arise in more complex scenarios.

### Making the Simple Complex

Up to this point, the development of our todo app has been agile alright, but we never had to support offline-capable clients or multiple clients that could run different versions of the software. That's about to change. Let's talk about offline first.

Currently, our todo application will simply stop working when the internet connection is down. Instead of the app, users will get a message that they are offline. Or - perhaps even worse - if the connection is unstable they will not even get any notification but they might simply see a white screen while some request is underway and they will wait and hope for a response and wait and hope for a response and wait and eventually get frustrated.

To fix this, we would like users to be able to access the app and perform all the relevant CRUD operations on todo items even if the internet connection is not reliable. Let's make this a feature request:

```cucumber
As an application user
I want to edit todo items even when my internet connection is unreliable
so that I can plan my life without having to worry about network quality.
```

This could be done by building full-fledged desktop or native apps or, to start <abbr title="¯\_(ツ)_/¯">simple</abbr>, by transforming the already existing web application into a *Progressive Web App* that can run in a desktop environment. In any case, all the relevant application data will have to be stored on the client so that it can operate in the absence of a network connection.

When a user can create or edit todo items even if the client is offline we need to provide a way to synchronize any changes once it comes back online. Luckily we have CouchDB on our team! There are a number of client-side adaptations like [PouchDB](https://pouchdb.com/) for browsers or [Cloudant Sync](https://www.ibm.com/analytics/us/en/technology/offline-first/) for phones that provide CouchDB-like storing capabilities for clients and implement the Couch replication protocol so synchronizing data between different parts of the system becomes simple and fun.

CouchDB is great when it comes to building offline capable apps and managing data synchronization. But clients that are offline may miss a software update. When this happens, there may be outdated apps that are still reading and writing data that adheres to older schemas. This problem becomes even more obvious once we build native apps that may have to be updated manually. So why not introduce them appropriately:

```cucumber
As a customer
I want to use the todo app on my iPhone and through a web interface
so that I can fit my workflow to my life and not the other way round.
```

If we provide users not only with a web app but also with native clients then updating the data schema to a newer version comes with certain additional challenges. Imagine someone is using our todo app both through the web interface and on their phone. When we release a new version of the software that includes a breaking change to the data schema the web app gets updated immediately once the page is reloaded. But the app on the phone may have to be updated manually, and this could happen now, or soon, or someday, or – never. In this situation, we will have multiple clients with multiple application versions in the system.

In this more complex scenario we are now confronted with three formidable challenges: we want to support multiple clients running multiple app versions, we want them not to break when the network connection is unstable, and we want to develop our software incrementally and stay agile, pivoting and releasing new versions in response to user demand. How could this ever work, you wonder? Let's wonder together.

### Traditional Migrations Fail

Why can't we just reuse the traditional migration strategy that has worked well before and update all existing documents on the server? This would mean that we picked a single point in time where we updated all documents on the server to be compatible with the new data schema. The changes would then be replicated to the clients' local databases.

This approach fails for a couple of reasons. Let's first consider the case where a web app may be offline thanks to our latest enhancements. Here's a chain of events that breaks the system: Haneen is using the web app, version one, in her browser right now. Because she is currently traveling through rural Germany she is editing todos in offline mode. She has just created a todo to pluck some flowers for her friend's birthday. Meanwhile, we release the new version of the app and migrate all existing data in the server-side database. When she's back at home, Haneen's app synchronizes the recent changes. First, it sends the new flowers-todo to the server-side database, but the document has missed the migration. Secondly, the updated documents get replicated from the server-side CouchDB to her in-browser database, so all her local documents get updated. This may take a moment but we can assume we find a way to verify the update is complete before allowing the app to resume. Eventually, the web app, which has all the latest features now, is ready to work. But what about the flowers-todo? Because it was created offline according to the previous version of the data schema, it has no `status` property and no `groupId` and is invalid according to the new schema. The new version of the app does not know how to deal with this old document. It might even break. Utter disaster.

#### Saving Traditional Migrations by Amending Them?

There is a way to address this particular problem, i.e. that we have performed a server-side migration and a few outdated documents are still around. The solution would have two parts:

1. The clients could *ignore* documents with outdated schemas. Old todo items could then no longer break the system.
2. A backend service could listen for incoming documents on the server-side database. If it encounters a document with an outdated schema it migrates this document up (assign to default group, set `status`) and after a while, it is replicated to all the clients.

From a user experience perspective this would mean that after the update Haneen's latest flower-todo would disappear at first – because it is outdated and ignored – but reappear in its new version a moment later once the migration and replication process is complete. There is a good chance this happens so fast that Haneen will never notice the gap.

As this discussion shows, there are times when you can get away with a traditional server-side migration by *amending* it with a continuous update of outdated documents. If you just want to provide a single offline capable web app, this strategy may just be what you need. But of course, our scenario is more complex because we don't want to restrict users to a single app. In fact, we want to provide web apps and native clients as well. This comes with additional difficulties.

If there are multiple clients with different versions operating on the same data, the traditional migration strategy cannot be saved anymore. To see why it fails, consider another exemplary scenario: Basti is using the app, version one, as a web app and he has also installed it on his iPhone. Today is migration day so we update the documents on the server-side database and prepare to hand out the new version of the app. When Basti reloads the website, he will get the new version of the app right away. But what about the iPhone app? Of course, all the updated data will be replicated to the iPhone's local database as well, but if Basti did not make sure to update the app immediately, it may not find any data to work with because we have effectively deleted all the old documents it is still depending upon and replaced them with new ones the app cannot handle. Whereas before we had new apps that are confronted with old data, we now have an old app that is not ready to handle new data. Utter disaster.

#### Saving Traditional Migrations by Moving Them to the Client?

Is it possible that this problem arises because we perform the data migration centralized on the server-side database? If we update documents on the server and then synchronize the changes to old apps, it's no wonder they are breaking. What if we instead ran migrations directly on the clients? The setup could look like this: once a client updates to a new version, it pauses for a moment and runs the migration on its local database. When this is finished, it resumes operation and now works with the new data. An old app would simply not run the migration and continue to work with the old data.

This will fail as well, however, and for a very similar reason as before. Remember that we are dealing with a distributed system that communicates through CouchDB's replication mechanism. In the current scenario, this would entail that the document updates that have just happened on a single client will be distributed throughout the system and will eventually update and overwrite documents on older clients as well. An older client may refuse to run the migration, but the documents still get updated through the replication mechanism. In the end, the old app breaks because the migration one client removed the required `isDone` property that was needed by the other. Basti's web app just broke his iPhone app.

The traditional migration approach, server-side or client-side, with or without a continuous update amendment, cannot be successful in our more ambitious scenario. We have to get creative and look for alternative solutions.

### Searching for Alternatives

When we introduced the `isImportant` flag to our data schema we mentioned only passingly that old documents without this flag would still be valid. This was because an app could treat a missing flag as `false` and the todo item as not important by default. We didn't need a big migration for this change at all because by interpreting the existing data appropriately the *clients* were able to ensure the smooth introduction of a new feature. Perhaps we can take cues from this pleasant experience and have the applications themselves play a bigger role in data migrations.

The examples that led us to think about migrations proper were the introductions of the `status` attribute and the grouping functionality which changed the todo items and called for a new document type. During the migration process we had to choose a correct `status` and assign existing todo-items to some default group, and potentially create the default group document in the first place. Perhaps all of this could be the responsibility of the apps instead! When a new application encounters a todo without a group, it can assign it to the default group *on the fly*, as it were. And when it encounters a todo without a `status`, it can look at its `isDone` property and infer the status, just as the server-side migration would. This way, the introduction of a new schema would not immediately break old apps.

<figure class="diagram" id="figure-2">
  <img src="./images/live-migration.svg" alt="Schematic view of adapter migration" />
  <figcaption>
    <b>Figure 2: Adapter Migration.</b>
    <span>
      An adapter enables a client to read documents of older formats. When it comes to persisting them the app will store the documents in their updated version.
    </span>
  </figcaption>
</figure>

Building this kind of functionality right into the application will become tedious and hard to maintain, though. So instead, let's write an *adapter* as an extension to the app that performs the migration. We will refer to this approach as an *adapter migration*. [Figure 2](#figure-2) illustrates the concept in broad strokes: an adapter is provided to update the old (white) document type to a newer version which the new app knows how to handle.

Adapters would enable newer applications to read older documents. And after a document has been read, the app could write it back to the database in its updated form. The migration, which previously happened during one big event would now happen step by step, whenever a client actually needed to handle a document. For lack of a better term, we could call this *eventual migration* to be in line with CouchDB's central concept of *eventual consistency*.

To elaborate on this approach a little further, let's take another look at the todo app. Our example application is live and we have todo items in the system. We now want to release a new version of the app that works with new document versions. The new application can read and write these new documents, and in order for it to process older documents, we provide it with an adapter that takes in old documents and returns new ones. For instance, an adapter could take in a todo item with an old schema:

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818",
  "schema": "todo-item",
  "version": 1,
  "title": "Calculate the carbon footprint of a bitcoin transaction",
  "isDone": true,
  "createdAt": "2017-11-01T00:00:00.000Z"
}
```

And it could return the updated document, a todo item that has been assigned a `status` and a default group:

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818",
  "schema": "todo-item",
  "version": 3,
  "title": "Calculate the carbon footprint of a bitcoin transaction",
  "status": "done",
  "groupId": "c21f969b5f03d33d43e04f8f136e7682",
  "createdAt": "2017-11-01T00:00:00.000Z"
}
```

From this point on the app knows how to proceed. Using the adapter, it can treat older documents just as if they were new ones.

Adapters have their own issues, though. Who would have thought! First of all, if we update documents once we *read* them, we will encounter massive amounts of conflicts throughout the system. This is because there can be multiple clients that read documents at the same time, e.g. when they display lists of todos on different devices. But if new versions of documents are created in several places at the same time this creates conflicts when the updates get replicated across the system.

To avoid those conflicts, old documents that are read through adapters should only be changed in the database when their content changes, in other words: the actual migration should happen *on write*. But even then we could get into trouble if an updated client starts migrating documents while there is still an older client that depends on the older documents. We seem to have gotten nowhere so far, only that we now have to maintain a potentially large number of adapters between who knows how many different schema versions that have to work on lots of different clients.

It seems like the basic problem with all approaches so far is that we try to maintain *multiple* client versions that are all supposed to work with a *single* version of the data schema. And how could this work in the first place?

So is there a way to maintain multiple versions of the data schema in parallel? CouchDB will certainly not object if the same information is stored in different formats - as long as the `_id` attributes of the respective documents are unique. Maybe there is a way to run *non-destructive migrations* then. For instance, adapters could read old documents and later store the information in a new document with an appropriate `_id` without overwriting the existing documents. This way, old apps won't break immediately. But if we don't make sure old documents reflect the latest changes from user interactions, how can old apps still be up to date when they are using old documents? And can they even deal with the new documents that can arrive from newer clients? Or is there a way to ignore those? And what if managing the `_id` gets too messy? We said databases are lightweight and cheap, can't we just have a completely new database for every version of the data schema?

Wow, this investigation has gotten out of hands! It looks like we need a more structured approach to get a grip on this messy collection of problems. But rest assured that our efforts will pay off soon. We've seen some strategies fail, but we also understand better what we are up against. Plus we have collected a number of ideas and built a few tools in the process. Let's switch gears now and approach our discussion with a tad of rigorous discipline.


## A Taxonomy for Reasoning about Migrations

When it comes to schema migrations our task is to modify data of a certain format so that it matches new format requirements. This process is usually destructive: the old data is assumed to be obsolete after the migration and is eliminated during the migration process. But we are now dealing with a situation where multiple schema versions may have to exist in parallel because there can be multiple clients in a system that run different versions of some application software.

It is time to approach this issue in an orderly fashion. To this end, we will propose a taxonomy for thinking about migration strategies that will take into account (1) where data is stored, (2) when it is being modified and (3) where this modification happens. This taxonomy is a kind of elaborate problem exposition and not yet a solution. We do have an opinion as to which approach works best in the scenario we have sketched out, but we are going to argue for that separately. For now, our goal is to take seriously the difficulties inherent to the migration problem, to lay out different angles for tackling those, and, most of all, to invite you, kind reader, to join us in a broader discussion of these issues.

### Where Is Data Being Stored?

Up to this point, we have only talked about setups where one user has a single database for all their data. In our running example, this database has one server-side replica and multiple replicas on the client devices, but it is still a *single* database. In the migration strategies we have seen so far, there existed only a *single* version of a document at a time as well. When it came to dealing with offline-capable clients and maintaining multiple app versions, the traditional approaches failed exactly because there were document versions missing that some applications still depended upon.

Maintaining multiple document versions in parallel can be achieved in two ways. First, multiple document versions can be stored in a *single* database *simultaneously*. The fact that the `_id`-attribute of a document must be unique means that we would have to adapt this property accordingly, for instance by keeping track of the version in the `_id` itself. As an example, our todo item documents could have `_ids` like `todo-item@1:ce2f71ee7db9d6decfe459ca9d000df5` and `todo-item@2:ce2f71ee7db9d6decfe459ca9d000df5` that carry information about the respective schema version. If we manage `_id`s carefully, it is possible to work with multiple document versions in a single database. Alternatively, each version of the data schema could have its own database. Instead of the standard *db-per-user* approach, this would imply that we set up multiple *dbs-per-user*, one for each schema version.

These new options, together with the traditional approach, form the three values along the *where-to-store-the-data* dimension, as we have basically three options available, which we will try to furnish with telling labels:

1. **Single-version**: Use a *single* database and keep only a *single* version of documents.
2. **Multi-version**: Provide a *single* database but have *multiple* document versions in it.
3. **Multi-db**: Set up *multiple* databases, each one storing documents that belong to a *single* schema version.

These three alternatives constitute the first dimension of the taxonomy we are building up. Next, we turn our attention to possible migration mechanics.

### When Is Data Being Modified?

From our previous discussion, we can identify two different mechanisms to perform migrations. The first one is the traditional approach where all data is modified whether or not this is actually required. We have seen that this could happen in a *one-shot* fashion at a single point in time where all existing documents are updated at once, or *continuously* while new documents are coming in. The essential tools to implement this type of migration are *tasks* that can be executed by the responsible services or workers and *hooks* that make it possible to intercept the data flow and perform document updates as soon as possible.

We have also already encountered the second migration mechanism in the form of adapters that were briefly discussed in the last section. In contrast to migration tasks, adapters perform migrations *on the fly*. It is characteristic of this approach that document updates happen only when documents are actually needed by an application. Data that is never touched will never be migrated. To get a handle on this *how-to*-dimension we propose to use the following terminology:

1. **Eager**: Migration strategies that perform document updates immediately for all available data.
2. **Lazy**: Strategies that only update documents if necessary.

A major difference between the two approaches is that lazy migrations have to intercept the read (and possibly the write) process of the applications while eager migrations can happen independently of any application behavior.


### Where Is Data Being Modified?

As a third and last dimension of a useful taxonomy, we suggest taking into account whether migrations are performed on the server-side database or on the client. Traditional migrations are only run on the server, but we have already started to consider alternatives in the previous discussions.

Running migrations on client-side databases comes with additional difficulties. For instance, it will be harder to fix bugs and maintain code that is distributed across multiple user devices. Performing migrations on a single server that you have direct access to makes migrations a lot easier to test, control, and possibly even revert. Additionally, multiple client applications may be written in a number of different programming languages so that the respective migration code, which can already be challenging to get correct in the first place, would have to be written multiple times in multiple languages. Lastly, the different clients that implement the [Couch replication protocol](http://docs.couchdb.org/en/master/replication/protocol.html) (like CouchDB, PouchDB, Cloudant Sync, etc.) do not generate the same revision ids (there are some plans to unify this behavior, though). Without consistent `_rev`s, creating the same document on multiple clients in parallel could lead to a massive amount of conflicts. This is not to say that running migrations on the client is a bad choice, we just want to caution that it may come with additional baggage.

To round things off, let us suggest a terminology to work with the distinction we have drawn here:

1. **Server-side**: Migrations are performed on a central server-side database and changes or updates are only later replicated to client databases.
2. **Client-side**: Migrations are run directly on client-databases, potentially in parallel, and changes will then be replicated through the system from each client to every other.

We have now established three dimensions of a taxonomy to help us reason about migration strategies. We could even view them as a kind of *orthogonal basis* of the space of possible migrations because we can move along each axis without affecting the other. This also means that our migration options multiply: we have identified three storage alternatives, two migration mechanisms, and two places where migrations could be performed for a total of *twelve distinct migration strategies*.

Even though we cannot discuss these options here we have built up a vocabulary for communicating about the different approaches. We can, for instance, distinguish the traditional migrations, which are *eager server-side single-version migrations*, from the *lazy client-side single-version migrations* we have named *adapter migrations* in the previous part. Equipped with our new tools, let's talk a little bit about what to keep in mind when developing new migration strategies before we wrap up this article with a summary of the things we've gone through.


## Developing Migration Strategies

> "The internet is broken" - Andi Pieper

Schema migrations across distributed systems are hard, and designing migration strategies is challenging for a lot of reasons. We have discussed some of the pitfalls and impasses you might run into, and there are a number of tradeoffs to consider as well. In this last section, we would like to collect some of the learnings from our discussion and address several open issues to keep in mind when deciding how to proceed.

#### Tradeoffs

Every software system is built in a particular context with specific requirements and resources that are limited in one way or another. When designing migration strategies it is important to keep these resources and requirements in mind. We will probably not be able to get the best of all the many worlds, we will not find the one, single, optimal migration strategy because all the different approaches have some merits and some disadvantages. Instead, we will have to make design decisions, and when we do, we want to keep some of the following points in mind.

*Legacy app support.* Single-version migrations can never support clients that require multiple versions of the schema. This point seems almost too obvious to state, but it is crucial to decide from the beginning whether you need to support legacy clients or not. Maybe limited offline capabilities are enough for your project. Or maybe it is better to find a way of enforcing client updates. Keep in mind, though, that if you are planning on dropping old versions at one point, applications need to be equipped with some graceful shutdown or update mechanism *from the very beginning.* This topic is larger than it might appear at first glance. Legacy support can mean different things, for instance, that you make sure old apps will not break when data is migrated, even though you might achieve this by ignoring some documents. It can also mean that you migrate data *down* so that older apps can have full access to all the latest information. What legacy support means depends on the respective context.

*Seamless migration processes.* Some of the strategies we have seen, e.g. the *on the fly* adapter migration, do not require any system downtime to run. This can be achieved by updating documents and distributing them across the system *before* the applications working with them get updated. This works only for *multi-version* migrations, of course. If keeping multiple versions of data helps allows for *zero downtime migrations*, we might even investigate if this strategy could be interesting outside the realm of document databases.

*Simplicity.* Some of the strategies we have seen are appealing because they are a lot easier to implement and maintain than others. If your resources are scarce, you might value simplicity higher than some of the more advanced features that require more elaborate migration processes. For instance, fixing bugs in migration code that is written in multiple languages and has already been shipped to clients which are currently offline can turn out to be a maintenance nightmare.

*Deduplication.* Storing documents for data-heavy applications can become expensive. Even more so when there are multiple versions of documents stored in parallel. *Multi-db* migrations, in particular, will run into this issue because they do not reuse documents across different versions of the data schema. The whole point of *multi-db* migrations is to have all documents necessary to support some app version in one dedicated database. So there might be multiple versions of the very same document stored on a user's device. On the other hand, it is ridiculously easy to delete old versions of the data schema: simply delete the corresponding database. This is how the problem of deduplication is related to questions of legacy app support and the purging of obsolete documents.

*Conflict safety*. We have seen how migration processes that update documents in many places, e.g. on multiple clients in parallel, can create conflicts. Depending on your project you might be able to come up with very efficient conflict resolution mechanisms or you decide that the number of occurring conflicts will be manageable. But keep in mind that, at least for CouchDB, larger amounts of unhandled conflicts in a document's history will lead to severe performance penalties.

#### Good Practices

Throughout our discussion, we have collected a number of recommendations and good practices when working with schemas and document databases in general and CouchDB in particular. Let's remind us of some of these learnings before concluding our discussion.

*Make your schema explicit!* We strongly recommend using JSON schema or some related vocabulary to formulate expectations about the form and shape of your data clearly.

*Use semantic versioning for your schema.* Knowing whether or not changing the document format breaks existing behavior is crucial when it comes to deciding which documents need to be migrated. However, remember that figuring out which changes are breaking depends on the context and requirements of your project.

*Whitelist document types.* Applications should not break when they encounter document types they don't know. This way, you are free to create new types without introducing a breaking change.

*Ignore unknown attributes.* For a similar reason, applications should be able to ignore document properties they don't need to know about.

*Prepare for dropping legacy support.* If you plan to drop support for legacy apps at one point, give them a chance to know they are outdated. If old apps know what to do when they are no longer supported gives them a chance to handle the transition more gracefully.

#### Stay in Touch

Although we have said what we had to say here, the real work is only just about to begin. Our main goal has been to open up a discussion about a really hard problem, to prepare the ground for exploring different solutions to it, and to illustrate some of the intricacies and complexities of the situation we are confronted with.

From here on out, our task is to design migration strategies for different use scenarios and to share and discuss them. We are planning to present you an *eager, multi-version, server-side migration* in the near future, which we think is both versatile and maintainable and we are looking forward to having a fruitful exchange about that.

For now, we'd like to thank you cordially for your attention and for following us through some rough terrain. We hope you feel like you've learned something. Should you feel inclined to leave any comments or suggestions, we would be delighted to receive [issues](https://github.com/jo/sync-tank/issues/new) and [pull-requests](https://github.com/jo/sync-tank/edit/master/distributed-migration-strategies/index.md) to our [project repository](https://github.com/jo/sync-tank).

Finally, we'd like to thank [immmr](https://www.immmr.com/), and in particular Marten Schönherr, for giving us some time to explore the issues presented here. We also appreciate the ideas and suggestions we received from the people who have already been part of the discussion and who helped us understand the migration problem better: Andi Pieper, Benjamin Kampmann, Jan Lehnardt, Martin Stadler and others.

---

Sync Tank, including [this article](https://github.com/jo/sync-tank/blob/master/distributed-migration-strategies/index.md), is [hosted on GitHub](https://github.com/jo/sync-tank). We are [@tfschmiz](https://twitter.com/tfschmiz) and [@MatthiasDumke](https://twitter.com/MatthiasDumke) on Twitter.
