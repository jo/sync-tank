---
layout: default
permalink: /distributed-migration-strategies/
---

# [Sync<br/>Tank](/) Distributed Migration Strategies
## How to handle schema changes in CouchDB
{:.no_toc}

## Table of contents
{:.no_toc}

1. this unordered seed list will be replaced by toc as unordered list
{:toc}

## Introduction

> "Software development is change management" - Ashley Williams, [A Brief History of Modularity](https://www.youtube.com/watch?v=vypCsVm5z28) at JSConf EU 2017

There is a certain kind of software that doesn't change over time. Dead software. Software that no one is using. If anybody would be using it, there would inevitably be suggestions for improvement and change requests. This is true for your pet project as it is true for enterprise software. Good software is constantly changing. That's the first part of our problem. The second part is the sort of software we would like to be writing: offline-capable, decentralized and scalable systems that can support and synchronize a whole range of different clients.

There are solutions and best practices for both of these challenges. Agile methodologies integrate readiness for change into the very process of developing software, allowing us to have working software in front of customers early on and adapting it based on feedback. There also exist tools and technologies that support offline-capable multi-client applications. But as it turns out it is not so easy to bring those requirements together. The cause of all the trouble is one of those parts of the system that is usually friendly and not as attention-craving as shiny front-ends or massive costs of operating server clusters: the data schema. The data schema?

Here's the short version of the problem: if an application implements new features, it often requires new data or data to be in different formats. That's a schema migration. Now if you have multiple applications that share data accross the system, even more so when some of those could be offline, it is impossible to update all or them at that same time. This leads to our initial conundrum:

* How can we update the data schema to support new applications when there might be old applications around that still rely on older versions of the schema?

At the time of this writing, performing schema migrations in distributed systems is largely unexplored terrain. In what follows, you will find a detailed exposition of our thoughts on this problem. We will outline various solutions from simple to more complex ones, and building up to a taxonomy of what we call *distributed migration strategies*.

To make the problem more managable, and because our professional background naturally leads us to it, we will focus our discussion on *Apache CouchDB*, especially when it comes to providing examples for the topics we discuss. And while we're at it, let us add a note on our background. We, Johannes J. Schmidt and Matthias Dumke, are data architects at a company called [immmr](https://www.immmr.com/) that builds communication applications for a variety of platforms. Our main task is to manage data sychronization accross a large, distributed system of native clients, web applications, desktop apps, and backend services. In this context, CouchDB is our main tool when it comes to providing stable and scalable replication of user data and allowing the company to stay agile.

In this article, we will begin by going through some basic concepts before we will illustrate the problem of schema migrations, starting from simple scenarios and integrating more complex requirements as we go along.


## Preliminaries: CouchDB, schemas, and migrations

Going through a technical discussion will be more fruitful if everybody is on the same page from the beginning. This is why in this part we would like to address a handful of general topics including a refresher on CouchDB, our understanding of schemas, migrations, and distributed systems, as well as best practices for working with data schemas in the wild. If you are already familiar with these topics, feel free to skim or skip this section. We will begin out discussion of migration strategies in the next part.

### Who is CouchDB?

As we just mentioned, and as the title of this article already suggests, we will narrow down the focus of our discussion to software systems backed by CouchDB. In case you didn't know,

> "Apache CouchDB™ lets you access your data where you need it by defining the Couch Replication Protocol that is implemented by a variety of projects and products that span every imaginable computing environment from globally distributed server-clusters, over mobile phones to web browsers."
>
> From the official [Apache CouchDB Documentation](https://couchdb.apache.org/)

We choose to focus on CouchDB because first of all, there is basically no production-ready alternative for a cross-platform data storage that provides synchronization capabilities and is also open source. Check out this [comparison chart](http://offlinefirst.org/sync/) to get an overview of offline capable storage options. And secondly, and pragmatically, we already know a lot about working with CouchDB, so this seems like a reasonable place for us to begin the discussion.

CouchDB is a scalable document-oriented store that allows efficient MapReduce data queries via indexing. It is a mature open source project under the Apache foundation that supports clients on all major platforms including web browsers, Android and iOS devices. One of its main features is reliable synchronization of data between different installations via multi master replication (as opposed to master slave replications). We do not assume in-depth familiarity with CouchDB though some expose will be helpful. In the following we will quickly go over the most important concepts that are relevant in the context of schema migrations. If you need a more in-depth introduction to the system, [CouchDB. The definitive Guide](http://guide.couchdb.org/) is a good place to start.

To begin with, let's get some technicalities out of the way: CouchDB provides an HTTP API for all database interactions like storing or retrieving information and managing databases. A single database within the system is just a document that can be created via a simple HTTP request like so: `PUT $COUCH/db-name`. This means that databases are very lightweight and can be created and deleted with ease. As a consequence, and as we shall see below, it is a common practice to create a new database for every user in a system (this approach is called 'couch-per-user' and we will make use of it here, though it is [not beyond dispute](https://medium.com/ibm-watson-data-lab/cloudant-best-and-worst-practices-7ee2040da1b)).

Data is transmitted and stored in the form of JSON-documents that can have any valid format you want them to have, plus some attributes such as `_id` and others starting with an underscore that have special meaning. Here's an example document to make this more concrete:

```json
{
  "_id": "todo-item:02cda16f19ed5fe4364f4e6ac400059b",
  "title": "Moonwalk the dog"
}
```

This is a JSON-document storing information about a todo item. We will see more of these documents when we start developing an example application in the next part. But for now we want to focus just on the `_id` attribute, the document identifier.

CouchDB uses `_id` to uniquely identify documents. If there is any information you need to be unique throughout the database, put it into the `_id`. If you don't set one yourself, CouchDB will create a UUID for you and set it for you, but we recommend to take the opportunity and store more meaningful information here, in other words: make the `_id` semantic! In the example we store the document type along with a UUID. This allows us to quickly identitfy the kind of document we are dealing with if this id is returned e.g. from a query. It also allows us to extract documents by type via the handy [`_all_docs`](http://docs.couchdb.org/en/latest/api/database/bulk-api.html#db-all-docs) query. Finding a good strategy for creating `_id`s is one of the challenges when it comes to data design. We will touch upon this topic a cople of times further down the road.

As was briefly mentioned above, CouchDB makes it possible to retrieve documents by implementing the powerful [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) pattern. In particular, it allows us to create so-called 'views' that define which data should be returned when they are queried. CouchDB will then build indices to find the requested data quickly and efficiently. As of version 2.0.0 CouchDB also allows queries via [Mango selectors](http://docs.couchdb.org/en/latest/api/database/find.html) that are based on MapReduce and are very performant yet flexible. The ability to query efficiently is of central importance when it comes to data design but there is no need to go into further depth at this point because the discussion of migrations will not require that.

There is one more topic we have to address here if we are going to talk about migrations further down the road, and that is *conflicts*. Conflicts can arise when data is being synchronized between multiple instances of the same database. To get a feeling for when this may happen, imagine the following 'split brain' scenario: Our friend Andi has CouchDB running on two devices and he opens the database where he stores his favorite things. Both instances are in the same state initially, and both contain the same document:

```json
{
  "_id": "favorite-programming-language",
  "_rev": "1-acabacabacabacabacabacabacabacab",
  "name": "Elixir"
}
```

Notice the `_rev`-attribute in this document. It's another one of those that have special meaning to CouchDB. Its value consists of two parts, the 'revision number' and the 'revision id', and the format is always `[revision-number]-[revision-id]`. This attribute is used to keep track of how a document changes over time. Every time a change happens, the revision number is incremented and a new revision id is assigned. Let's see this in the example.

If there was a network connection between Andi's devices, any changes he'd make to this document on one of them would be replicated to the other so that his data would be synchronized. However, let's assume there is currently no network connection. Now Andi goes and changes his favorite language on both devices to something different (in order to break the system, of course, not because his preferences are changing so rapidly). So here is the document as it is stored after the changes on his first and second device respectively:

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

In both cases, the revision number is increased to 2 and the name attribute has been changed, of course. Now if both devices are connected CouchDB will try to bring all data to the same state, but it is inherently unclear which one of the changes should determine the final version of the language document. Even if it was feasible to compare document write times accross different systems (which is often very hard to accomplish) and then choose a final document on a last-write-wins basis, it is not clear that this manner of handling conflicts is always the preferred way. This is why CouchDB will by default pick some version as winning - deterministically based on the revision id and number - and will reference conflicting versions in the document. This way, no information is lost and conflicts can be resolved more elaborately if necessary. For completeness, here is the final document that CouchDB automatically creates after default handling of the conflict.

```json
{
  "_id": "favorite-programming-language",
  "_rev": "2-bdec0dcd2c295a50d54556ebdca300cd",
  "name": "Lua",

  "_conflicts": ["2-4ab8a90d4c7c68221719559660a477ff"]
}
```

The next time you retrieve this document, you can instruct CouchDB to also include any conflicting revisions and it will include the `_conflicts`-attribute. Based on this it is now possible to implement mechanisms for resolving conflicts in ways that fit your needs best. The important point is that CouchDB will not lose any information, provides reasonable defaults, and still gives you full flexibility over how to resove the conflicts you encounter.

For the time being, these notes will suffice as a review of central CouchDB concepts. We will touch upon many of the points throughout the rest of this article. Next up is an explanation of what we have in mind when we talk about schemas, migrations, and distributed systems.


### Basic concepts: schemas, migrations, and distributed systems

If you want to store and retrieve data in an automated and efficient way, it is important to have some knowledge about the format or structure of this data. This is meta-information specifying things like data attributes and the data types to be used to store them. We would like to think of the **data schema** as all relevant structure-information about the different pieces of data to be stored by an application.

On this general account, a schema formalizes the structure of your data. Many database systems actually require in advance an explicit account of what the data to be stored is going to look like. PostgreSQL for example allows to store relational data where entries have to adhere to one of the [available data formats](https://www.postgresql.org/docs/8.4/static/datatype.html). These formats will even be validated on store-time.

Document databases like CouchDB are sometimes referred to as *schemaless*. This only means that the database will not force you to make your schema explicit before storing data, it does not mean that your data will not have a schema. In the end, a schema is not defined by the database anyways, but by an *application* expecting data to have a certain format. If you don't make your schema explicit, it will simply be *implicit*.

> "Usually when you're talking to a database you want to get some specific pieces of data out of it: I'd like the price, I'd like the quantity, I'd like the customer. As soon as you are doing that what you are doing is setting up an implicit schema."
>
> Martin Fowler, [Introduction to NoSQL](https://www.youtube.com/watch?v=qI_g07C_Q5I)

Thinking in terms of implicit schemas as defined by the application's expectations may require a change of perspective, but it will benefit us in the long run. Moreover, in an upcoming section we will recommend that schemas be made explicit regardless as this will allow us to be more precise about what some piece of data looks like and to keep track of how it changes over time.

Once the application evolves and starts to make new assumptions about the data, flexibility of the (implicit) schema becomes an issue. With a rather lenient document database like CouchDB it may be possible to simply store some additional information in documents at first, but after a while some documents will become outdated and unable to support the functionality new app versions require. At this point we will be forced to transform existing documents into a new form (whether we overwrite existing data, transform it on-the-fly, or find some other way to perform those updates remains a question to be discussed). Changing existing data to adhere to new requirements is called a **data migration**.

The last concept to address are **distributed systems**. According to the Wikipedia,

> "A distributed system is a model in which components located on networked computers communicate and coordinate their actions by passing messages."
>
> Wikipedia, [Distributed computing](https://en.wikipedia.org/wiki/Distributed_computing)

To apply this broad definition to the CouchDB environment, we can interpret database documents as messages, data transfer via replication as message passing, and the various apps on phones and in browers and the backend services as the components located on networked computers. This can mean that data will have to be stored not only on a server but also on user-devices, which in turn must be enabled to synchronize data updates with the rest of the system so all parts can operate on a shared informational basis.

In this distributed scenario, different applications can make very different use of the same data. For instance, an Android app may use data to display a list of todos to the user while a backend service may be interested in the metadata to build usage profiles. We therefore have software that accesses, processes, and stores data in very different locations and according to their very different requirements. For the whole system to be intact it is mandatory that the structure of the data not be changed in any unforseeable way. The different parts of the system, and consequently the different teams working on the different parts of the system, are bound by an (implicit) contract, by the (implicit) data schema they all have to respect. We may even go one step further and say that *the data schema is the effective database API* of the distributed system because it ultimately defines the way in which data can be accessed. From this perspective, a schema-change implies an API change for any part of the system that is dependent on the data. This is why it is of such importance to have a strategy for dealing with schema changes.

When migrations become necessary in distributed systems, we run into the complex issues this article is about.


### Make your schema explicit!

We have already mentioned that even if people may call your database *schemaless* this does not mean you can actually work without a data schema. Granted, you may be able to change your mind at any time and store differently structured information and the database will be fine with that. This is why schemaless databases have a reputation for facilitating flexible development. But at the very least the data schema will be *implicit* as your applications have certain expectations about what kind information there is in the database. We have found that it is very helpful to be more explicit about the schema. Allow us to argue for that:

1. A data schema that is made explicit makes **communication** about data a lot easier. This is true across time (when you come back to your project three months later) and across teams (when the web team is supposed to add the same functionality to their apps as the Android team).
2. An explicit schema also allows for format checking and **schema validation**, which can expose bugs in applications earlier, and in general lead to more reliable and consistent data. More reliable and consistent data makes application development not only easier, but practically feasible in the first place.
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

The example features `schema` and `version` attributes on the document with values of `todo-item` and `1` respectively. This will be most valuable information once we have to implement schema migrations. We also recommend to maintain a schema version for each document type *separately*. The version is then tightly coupled with the data format, while leaving enough room to change document versions independently of each other. For instance, we could upgrade `address` documents from version `1` to `2` while the related user data could still be stored in `user` documents of version `1`. This prevents us from having to increase schema versions unnecessarily.

Being explicit about schemas is also the first step to *versioning* them. As a final practical recommendation we suggest to use [semantic versioning](http://semver.org/), properly interpreted to work well with data schemas, in order to keep track of how a schema evolves. We use semantic versioning to determine the version of the *database schema*, which is the collection of all *individual document schemas*. This way, it is easy to understand the type and severity of schema changes:

* **Breaking changes** happen every time old documents do not validate against a new schema anymore.
* **Features** are all additions and enhancements that do not affect the validity of existing documents.
* **Fixes** are small improvements like correcting typos in schema descriptions.

In the above example, the schema specification for the document has its own version, `1.0.0`, which will be updated as the schema evolves. With versioning in place on the document level it is only a small step to creating a schema manifest that bundles and specifies the document versions that make up a schema version. To be concrete, we could use a good old `package.json` to specify that user, address, and todo item documents are the constitutive parts of the database schema for an examplary todo application.

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


## If it ain't broke, don't fix it

There are times when updates to a data schema do not endanger the functionality of a system. In these scenarios, migrations are either straight forward or not even necessary in the first place. We would like to start by looking at some such cases because if you can get away without the multi-client multi-version migration hassle you can save yourself a lot of work. In particular, we will start by looking at unobtrusive additions to the data schema as well as traditional server-side migrations.

### Setting the stage: A toy problem

While this article is not supposed to be a tutorial, it will be instructive to have a working example at hands to illustrate some points. Our example of choice is the 'Hello World' of software applications, the todo app. A simple todo app does not look like the most daunting challenge to face from a data architecture perspective: todos can have titles and texts, maybe a creation date and a 'done' flag. None of this would make you sit down and write a full article about different strategies for how to accommodate this information in your database. But things get a lot more interesting once we agree to meet a few additional requirements:

1. Users should be able to edit their todos even when their internet connection is unstable or down, and the application should synchronize all changes once it reconnects. In other words: the app should be **offline-capable** (what exactly this means will become clearer as we go along).
2. The application development should be **agile**, meaning that we want to support frequent changes to all aspects of the system, including the data schema.
3. Users should be able to use the todo app on **multiple clients**, for instance as a web service and as an iPhone app, with potentially different versions of the software running on different clients.

These requirements will eventually lead us to think about distributed migrations. Of course, this is all way ahead. For now we just want you to know that a tiny todo app can cause a whole lot of trouble further down the road. But this section is about starting simple, so let's start with a simple setup for a simple todo app and take on new challenges as we go along.

In the simplest of todo app scenarios, we want to enable a user to manage a few, or maybe a few thousand todo items from the comfort of their web browser. The basic building blocks to set up such a service are: a web app to provide a nice interface, a server to deliver that application, a central database that stores all those todos, and bit of infrastructure to glue the pieces together. We have decided to use CouchDB as our database and as we mentioned above we chose to set up the system such that every user gets their very own database. This is a common practice in the CouchDB world that ties in well with the fact that CouchDB permissions-management is modeled on a per-database basis. It entails that there is no way for one user to get direct access to anybody else's data because they are stored in different databases. Everybody manages their own todos in their own database. And yes: once our product goes viral and there are two million users there will be two million databases. Well, Ops problem. In fact, Ops will be happy to find that CouchDB comes with clustering abilities so scaling up is not a terrifying prospect - unlike when you run out of space with your relational database (though, to be fair, management may not be as happy, because operating a CouchDB cluster at scale may become a somewhat costly affair).

For now, this piecemeal approach of worrying about one user and one database at a time simplifies our problem. To complete the first step and bring version one of our todo app to the market, all there is to do as far as the schema is concerned is to decide how a single todo item is supposed to look. And since we wanted to start simple, and since the *sine qua non* of a todo item is basically just a title and a flag, here's an example of the first, launch-ready version of a valid todo item document to be stored in a user's CouchDB that also adheres to the schema we specified in the last part:

```json
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "schema": "todo-item",
  "version": 1,
  "title": "reimplement my pet project in Rust",
  "isDone": true,
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

### The world is changing: new requirements

The first weeks have passed, marketing has done a great job and our app is quite popular, especially with young professional single parents in urban areas. Feature requests are coming in and a decision is made to enhance the product. So we face a new requirement:

```cucumber
As a web app user
I want to mark a todo as important
so that I can find it easier.
```

Obviously the data schema will need some enhancements in order to store that new information. In particular, we will want to add an `isImportant` flag to each todo item. This change is rather unobtrusive because it leaves already existing todo items intact: since existing documents will not have the `isImportant` attribute, we can simply treat them as not important by default. All we have to do is make sure that new version of the app will be able to handle missing `isImportant` flags.


This change was not too hard to implement. A second request that many users have made is the ability to change the color theme of their app:

```cucumber
As a web app user
I want to choose between different color themes
so that I can express myself by personalizing my tools.
```

Obviously, this feature has no direct implications for structure of todo items. We decide to introduce a new document type, a `settings`-document, that will store the color information and perhaps other general preferences that will come up in the future. Since we only want one global settings document, we can set the `_id` to something simple.

```json
{
  "_id": "settings",
  "schema": "settings",
  "version": 1,
  "color": "#e20074"
}
```

As with the introduction of an `isImportant` property, the new settings document is unobtrusive in the sense that it does not break existing applications. We could even create those new documents before the web app is ready to work with them. In general, creating documents with unknown types should not affect applications. For CouchDB, which relies heavily on views to retrieve data, this behavior can be achieved by *whitelisting* documents at the access level. In other words: when building up a view, confirm for each document that it's type is known, and ignore it otherwise. This way, new document types can be added without affecting existing behavior.


So far we have amended the todo item schema and introduced a new document type. Both operations are non-breaking feature changes to our data schema according to semantic versioning. But now let's look at yet another feature request that will have a deeper impact on our data format:

```cucumber
As a web app user
I want to assign each todo item to a group
so that I can isolate tasks that belong together in order to be more focussed
```

There is a straight-forward and time-proven way to solve a grouping task like this: *one-to-many associations*. We can create a document for each group and then link todo items to groups by storing a group identifier along with each todo item. The identifier that establishes this link is sometimes referred to as a *foreign key*. Before looking at the schema implications of this approach, we would like to add a more general note on associations in document databases because we have seen how this topic can trip up people new to the technology.

*Associations* describe the connection of entities: *one-to-one*, *one-to-many*, or *many-to-many*. In *relational database design*, associations are typically established via *foreign keys* as described above. This pattern is so common that people sometimes believe it actually defines what a relational database is in the first place and they are surprised to see it appear in other places as well: "How can there be foreign keys in CouchDB documents? I thought it's not a relational database!" But associations are not what makes relational databases relational – *relations* are. A [relation](https://en.wikipedia.org/wiki/Relation_(database)) is a set of tuples of values with pre-defined types. That's all. There are operations like *selections* and *joins* defined on these relations (cf. [relational algebra](https://en.wikipedia.org/wiki/Relational_algebra)) that enable the complex queries and powerful query optimizations we all know and love. Associations play an important role in those operations, but the general concept goes far beyond relational databases themselves. Linking entities by their identifier is a very versatile strategy of data design, as we shall see below. Long story short: *relations* and *associations* are two different pairs of shoes.

To prepare the `todo-item` and `group` association we first need to define `group` documents. Since we don't have any other requirements yet, let's simply give each group a name. Here's an examplary `group` document for illustration purposes, so we are all on the same page:

```json
{
  "_id": "group:e302f183a96194f7d19dce0eaf5e3cf8",
  "schema": "group",
  "version": 1,
  "name": "Things I also want to do"
}
```

In order to associate a todo item with a group, we can use the group document's id, which can be found in its `_id`-attribute. This way, a corresponding todo item could look like this:

```json
{
  "_id": "todo-item:4d3956b34778d92c676e3a487dad73fe",
  "schema": "todo-item",
  "version": 2,
  "title": "change the world",
  "isImportant": "true",
  "isDone": false,
  "goupId": "e302f183a96194f7d19dce0eaf5e3cf8",
  "createdAt": "2017-11-14T00:00:00.000Z"
}
```

The `groupId` attribute clearly establishes the association of the todo item with the group above. It is also easy to see how multiple todo items could belong into this group if each of them specifies the same group id.

After talking with developers about the direction that application development should take in the future we decide that we do not want to have documents without group id in the system. Every todo item *must* belong in exactly one group. To enforce this, we *require* the `groupId` attribute to be defined in every todo item document. This will allow us to *validate* the requirement and prevent us from storing documents without a group.

We now have a good idea about how to adapt the data schema in order to react to the latest feature requests. We have even found a way to make developers happy by enforcing their expectations through schema validations. At this point we would like to release a new version of the web app that allows users to sort their todo items into groups.

But wait! What about the todo items that have been created in the past? Those documents are now *invalid* because they do not specify a group id. When our new apps encounter such an older document they will not know what to do with it. It looks like we will have to change existing documents somehow. We could, for instance, associate them with some 'default' group. But whatever strategy we pick, there's no way to sugarcoat it: this is a *breaking change* to our schema and we need a data migration!


### Traditional server-side migration

We are now at a point where a new version of an application may be confronted with older documents in the system that it does not know how to handle. But this prospect does not scare us because there is a common practice that is used in this scenario, and one that you probably know well already. We will call this the *traditional server-side migration*.

<figure class="diagram" id="figure-1">
  <img src="/distributed-migration-strategies/images/transactional-migration.svg" alt="Schematic view of traditional server-side migration" />
  <figcaption>
    <b>Figure 1: Traditional Server-Side Migration.</b>
    <span>
      Introducing a new version of an application may require us to update existing documents (white) to a new version (green). This switch happens at a single point in time. Some documents (two-colored example) can be handled by both app versions and do not need to be transformed at all.
    </span>
  </figcaption>
</figure>

Since we have stored all our data in one central database, it will be easy enough for us to access all existing todo items and update them to adhere to the new schema. Ruby on Rails's way of handling migrations provides a very straight forward example of this approach. In Rails we would define a migration that formalizes the schema change (prepare the database to store the new `status`, move existing `isDone` information over into the `status`, remove the `isDone` field from the todo item table). We would then take the system down, run the migration (the famous `rails db:migrate`, formerly `rake db:migrate`) and hand out the updated application once the database is back up. If anything goes wrong during this process, there will be a rollback because the migration is wrapped in a transaction. During the process we will of course incur some downtime, but on the plus side we always have consistent and up to date documents and everyone will get the latest version of our application.

This same strategy works with CouchDB as well. Let us roughly sketch out the main steps to go through when performing a **traditional server-side migration on CouchDB**. Here's our recipe:

```
1. Set up a new, empty cluster (in case you want to revert or abort the migration at some point).
2. Switch to maintenance mode.
3. Perform a replication of all data from the old to the new cluster.
4. Now run a migration script to update data on the new cluster.

If a check verifies that migration was successful:
  5. Switch to the new cluster and switch off maintenance mode. Done.
Else:
  5. Rollback migration by deleting the new cluster.
```

[Figure 1](#figure-1) illustrates the traditional migration strategy. It shows how both the application and the documents are updated together in a single step. White documents can be handled by the white version of the app while the green app needs green documents. A special case is the two-colored document. There might be documents that do not need to change during a migration and that can be handled by multiple versions of the client. Think of the settings document we have added previously. Even if we have to change the structure of todo items this does not mean we have to change how the settings document looks.

This migration procedure is very common for the type of monolithic centralized setup we have described so far. It does not come without some [problems of it's own](https://about.futurelearn.com/blog/your-database-is-a-distributed-system) but overall it is a well established and reliable practice. Alas, this is not a viable solution anymore once we bring in some of the more challenging requirements we have omitted so far.


## Fail fast, fail often

Failure tends to lead to success. When you fail with one approach, you can quickly develop new insights into a problem, and you can free your mind and pivot and try out alternatives. This process will often produce better results than sticking with one initial strategy could ever achieve. So we are going to fail in this part. In fact, we are going to fail a couple of times.

As a first step we will bring back much of the complexity we have ignored so far. In this new context, we will be forced to look at more powerful migration strategies because our previous approaches are not viable anymore. In the spirit of this section, all the strategies discussed here will have serious drawbacks that should nevertheless prepare us well for a more systematic discussion that is coming up in the next section. Our primary goal for now is to develop a better understanding about the different types of problems that can arise in more complex scenarios.

### Making the simple complex

> "Document databases are really cool… until you have to make a breaking change to the schema. Then it feels like “good luck with all that!” :D"
>
> Ben Nadel [on Twitter](https://twitter.com/BenNadel/status/918604059304779776)

Up to this point the development of our todo app has been agile alright, but we never had to support offline capable clients or multiple clients that could run different versions of the software. That's about to change. Let's talk about offline first.

Currently, our todo application will simply stop working when the internet connection is down. Instead of the app, users will get a message that they are offline. Or - perhaps even worse - if the connection is unstable they will not even get any notification but they might simply see a white screen while some request is underway and they will wait and hope for a response and wait and hope for a response and wait and eventually get frustrated.

To fix this, we would like users to be able to access the app and perform all the relevant CRUD operations on todo items even if the internet connection is not reliable. Let's make this a feature request:

```cucumber
As an application user
I want to edit todo items even when my internet connection is unreliable
so that I can plan my life without having to worry about network quality.
```

This could be done by building full-fledged desktop or native apps or, to start simple, by transforming the already existing web application into a *Progressive Web App* that can run on a desktop environment. In any case, all the relevant application data will have to be stored on the client so that it can operate in the absence of a network connection.

When a user can create or edit todo items even if the client is offline we need to provide a way to synchronize any changes once it comes back online. Luckily we have CouchDB on our team! There are a number of client-side adaptations like [PouchDB](https://pouchdb.com/) for browsers or [Cloudant Sync](https://www.ibm.com/analytics/us/en/technology/offline-first/) for phones that provide CouchDB-like storing capabilities for clients and implement the Couch replication protocol so synchronizing data between different parts of the system becomes simple and fun.

CouchDB is great when it comes to building offline capable apps and managing data synchronization. But clients that are offline may miss a software update. When this happens, there may be outdated clients that are still reading and writing data that adheres to older schemas. This problem becomes even more obvious once we build native apps that may have to be updated manually. So why not introduce them appropriately:

```cucumber
As a customer
I want to use the todo app on my iPhone and through a web interface
so that I can fit my workflow to my life and not the other way round.
```

If we provide users not only with a web app but also with native clients then updating the data schema to a newer version comes with certain additional challenges. Imagine someone is using our todo app both through the web interface and on their phone. When we release a new version of the software that includes a breaking change to the data schema the web app gets updated immediately once the page is reloaded. But the app on the phone may have to be updated manually, and this could happen now, or soon, or someday, or – never. In this situation we will have multiple clients with multiple application versions in the system.

In this more complex scenario we are now confronted with three formidable challenges: we want to support multiple clients running multiple app versions, we want them not to break when the network connection is unstable, and we want to develop our software incrementally and stay agile, pivoting and releasing new versions in response to user demand. How could this ever work, you wonder? Let's wonder together.

### Traditional migrations fail

Why can't we just reuse the traditional migration strategy that has worked well before and update all existing documents on the server? This would mean that we picked a single point in time where we updated all documents on the server to be compatible with the new data schema. The changes would then be replicated to the clients' local databases.

This approach fails for a couple of reasons. Let's first consider the case where a web app may be offline thanks to our latest enhancements. Here's a chain of events that breaks the system: Haneen is using the web app, version one, in her browser right now. Because she is currently travelling through rural Germany she is editing todos in offline mode. She has just created a todo to pluck some flowers for her friend's birthday. Meanwhile we release a new version of the app and migrate all existing data on the server-side database. Back at home, Haneen's app synchronizes the recent changes. The updated documents get replicated from the server-side CouchDB to her in-browser database. This may take a moment but we can assume we find a way to verify the update is complete before allowing the app to resume. Eventually the web app, which has all the latest features now, is ready to work. But what about the flowers-todo? It was created offline according to the previous version of the data schema. It still has the simple `isDone` flag instead of the new fancy `status`. The new version of the app does not know how to deal with this old document. It might even break. Utter disaster.

This problem on its own can be dealt with, and the solution has two parts.

1. First we need to make sure that the new app does not break when it encounters older documents. This should not be too hard. It basically has to ignore the `isDone`-flag in this case. And the fact that a `status`-document is missing should cause no troubles for the application anyways because this could also happen under normal circumstances where documents simply need some time to be synchronized.
2. The second part of the solution is a backend-service that continuously updates older documents that might be coming in. In our example, the old flower-todo will be replicated to the server at some point. The service, which would look at incoming documents, would notice that the version is outdated and perform a *post hoc* migration. Then the new flower-todo is synced back to the clients. Problem solved.

As this discussion shows, there are times when you can get away with a traditional server-side migration by amending it with a continuous update of potentially outdated documents. If you just want to provide an offline capable web app, this strategy may be just what you need. Of course, our scenario is more complex because we want to provide native clients as well. This comes with additional difficulties.

If there are clients with multiple versions operating on the same data, the traditional migration strategy is lost beyond reclaim. To see why it fails so badly, consider another exemplary scenario: Basti is using the app, version one, as a web app and he also has it installed on his iPhone. Today is migration day so you update the documents on the server-side database and prepare to hand out the new version of the app. When Basti reloads the web site, he will get the new version of the app right away. But what about the iPhone app? Of course, all the updated data will be replicated to the iPhone's local database as well, but if Basti did not make sure to update the app immediately, it starts to show some weird behavior or simply breaks because we effectively deleted the old documents it was depending upon. Utter disaster.

Is it possible that this problem arises because we perform the data migration centralized on the server-side database? What if we did it directly on the clients instead? The setup could look like this: once a client updates to a new version, it pauses for a moment and runs the traditional migration on its local database. When this is finished, it resumes operation and now works with the new data. This will fail as well, however, and for a very similar reason. Remember that we are dealing with a distributed system that communicates through CouchDB's replication mechanism. In the current scenario, this would entail that the document updates that have just happened on a single client will be distributed throughout the system and will eventually update and overwrite documents on older clients as well. Basti's web app just broke his iPhone app.

The traditional migration approach, server-side or client-side, with or without a continuous update amendment, cannot be successful in our complex scenario. There is no easy way to say this: we have to get creative and look for alternative solutions.

### Searching for alternatives

When we introduced the `isImportant` flag to our data schema we mentioned only passingly that old documents without this flag would still be valid. This was because an app could treat a missing flag as `false` and the todo item as not important by default. We didn't need a big migration for this change at all because by interpreting the existing data appropriately the *clients* were able to ensure the smooth introduction of a new feature. Perhaps we can take cues from this pleasant experience and have the applications themselves play a bigger role in data migrations.

<figure class="diagram" id="figure-2">
  <img src="/distributed-migration-strategies/images/live-migration.svg" alt="Schematic view of adapter migration" />
  <figcaption>
    <b>Figure 2: Adapter Migration.</b>
    <span>
      An adapter enables a client to read documents of older formats. When it comes to persisting them the app will store the documents in their updated version.
    </span>
  </figcaption>
</figure>


The example that led us to think about migrations proper was the switch from the simple `isDone` to the more expressive `status` that changed the core todo items and required the introduction of a new document type. During the migration process we have to take existing todo items and find an appropriate mapping of an `isDone`-value to one of the new states. Building this right into the app will become tedious and hard to maintain. Instead, let's write an *adapter* as an extension to the app that performs the migration *on the fly* as it were. Let will refer to this approach as an *adapter migration* for the time being. [Figure 2](#figure-2) illustrates the concept in broad strokes: an adapter is provided to update the old (white) document type to a newer version which the new app knows how to handle.

Adapters could enable newer applications to read older documents. And after a document has been read, the app could write it back to the database in its updated form. This way, the transformation of all documents that previously happened during the big migration event would now happen step by step, whenever a client actually needs to handle a document. For lack of a better term we could call this *eventual migration* to be in line with CouchDB's central concept of *eventual consistency*.

To illustrate this approach with an example, let's take another look at the todo app. Our example application is live and we have todo items in the system. We now want to release a new version of the app that works with new document versions. The new application can read and write these new documents, but in order for it to process older documents, we could provide it with an adapter that takes in old documents and returns new ones. For instance, an adapter could take in a todo item with an old schema:

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

And it returns the updated documents, a todo item and a status-document with a plausible mapping from `isDone` to `status`:

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818",
  "schema": "todo-item",
  "version": 2,
  "title": "Calculate the carbon footprint of a bitcoin transaction",
  "createdAt": "2017-11-01T00:00:00.000Z"
}
```

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818:status",
  "schema": "todo-item-status",
  "version": 1,
  "status": "done"
}
```

From this point on the app knows how to proceed. Using the adapter, it can treat older documents just as if they were new ones.

Adapters have their own issues, though. Who would have thought! First of all, if we update documents once we *read* them, we might encounter massive amounts of conflicts throughout the system. This is because there can be multiple clients that read documents at the same time, e.g. when they display lists of todos on different devices. But if new versions of documents are created in several places at the same time this creates conflicts when the updates get replicated across the system.

To avoid those conflicts old documents that are read through adapters should only be changed in the database when their content changes, in other words: the actual migration should happen *on write*. But even then we could get into trouble if an updated client starts migrating documents while there is still an older client that depends on the older documents. We seem to have gotten nowhere so far, only that we now have to maintain a potentially large number of adapters between who knows how many different versions of the data schema that work on different clients, so they will probably have to be written in different languages.

It seems like the basic problem with all approaches so far is that we try to maintain *multiple* client versions that are all supposed to work with a *single* version of the data schema. And how could this work in the first place?

So is there a way to maintain multiple versions of the data schema in parallel? CouchDB will certainly not object if the same information is stored in different formats - as long as the `_id` attributes of the respective documents are unique. Maybe there is a way to run *non-destructive migrations* then. For instance, adapters could read old documents and later store the information in a new document with an appropriate `_id` without overwriting the existing documents. This way, old apps won't break immediately. But if we don't make sure old documents reflect the latest changes from user interactions, how can old apps still be up to date when they are using old documents? And can they even deal with the new documents that can arrive from newer clients? Or is there a way to ignore those? And what if managing the `_id` gets too messy? We said databases are lightweight and cheap, can't we just have a completely new database for every version of the data schema?

Wow, this investigation has gotten out of hands! It looks like we need a more structured approach to get a grip on this messy collection of problems. But rest assured that our efforts will pay off soon. We've seen some strategies fail, but we also understand better what we are up against. Plus we have collected a number of ideas and built a few tools in the process. Let's switch gears now and approach our discussion with a tad of rigorous discipline.


## A taxonomy for reasoning about migrations

The common approach when it comes to schema migrations is to modify data that currently adheres to one format so that it matches new format requirements. This process is usually destructive: the old data is assumed to be obsolete after the migration and is eliminated during the migration process. But we are now dealing with a situation where multiple schema versions may have to exist in parallel because there can be multiple clients in a system that run different versions of some application software.

It is time to approach this issue in an orderly fashion. We will begin the discussion by proposing a taxonomy for thinking about migration strategies that will take into account *where* data is stored, *when* it is being modified and *where* this modification happens. After that we will take an in-depth look at one particular approach that we believe works best in our current situation.

### Where is data being stored?

Up to this point, we were only talking about setups where one user has a single database for all their data. In our running example, this database has one server-side replica and multiple replicas on the client devices, but it is still a *single* database. In the migration strategies we have seen so far, there existed only a *single* version of a document at a time as well. When it came to dealing with offline-capabable clients and maintaining multiple app version the traditional approaches failed exactly because there were document versions missing that some applications still depended upon.

Maintaining multiple document versions in parallel can be achieved in two ways. First, multiple document versions could be stored in a *single* database simultaneously. The fact that the `_id`-attribute of a document must be unique means that we would have to adapt this property accordingly, for instance by keeping track of the version in the `_id` itself. As an example, our todo item documents could have `_ids` like `todo-item:ce2f71ee7db9d6decfe459ca9d000df5:v:1` and `todo-item:ce2f71ee7db9d6decfe459ca9d000df5:v:2` that carry information about the respective document schema version. If we manange `_id`s carefully, it is possible to work with multiple document versions in a single database. Alternatively, each version of the data schema could have its own database. Instead of the standard *couch-per-user* approach this would imply that we set up multiple *couches-per-user*, one for each schema version.

These new options, together with the traditional approach, form the three values of the *where-to-store-the-data* dimension, as we have basically three options available, which we will try to furnish with telling labels:

1. **Single-version**: Use a *single* database and keep only a *single* version of documents.
2. **Multi-version**: Provide a *single* database but have *multiple* document versions in it.
3. **Multi-db**: Set up *multiple* databases, each one storing documents that belong to a *single* schema version.

These three alternatives constitute the first dimension of the taxonomy we are building up. Next we turn our attention to possible migration mechanics.

### When is data being modified?

From our previous discussion we can identify two different mechanisms to perform migrations. The first one is the traditional approach where data is modified no matter if it is required or not. We have seen that this could happen in a *on-shot* fashion at a single point in time where all existing documents are updated at once or *continuously* while new documents are coming in. The essential tools to implement this type of migration are *tasks* that can be executed by the responsible services or workers and *hooks* that make it possible to intercept the data flow and perform document updates as soon as possible.

We have also already encountered the second migration mechanism in the form of adapters that were briefly discussed in the last section. In contrast to migration tasks, adapters perform migrations *on-the-fly* as it were. It is characteristic for this approach that document updates happen only when documents are actually needed by an application. Data that is never touched will never be migrated. To get a handle on this *how-to*-dimension we propose to use the following terminology:

1. **Eager**: Migration strategies that perform document updates directly for all available data.
2. **Lazy**: Strategies that only update documents if necessary.

A major difference between the two approaches is that lazy migrations have to intercept the read (and possible the write) process of the applications while eager migrations can happend independently of any application behavior.


### Where is data being modified?

As a third and last dimension of a useful taxonomy we suggest to take into account whether migrations are performed on the server-side database or on the client. Traditional migrations are only run on the server, but we have already started to consider alternatives in the previous discussions.

Running migrations on client-side databases comes with additional difficulties. For instance, it will be harder to fix bugs and maintain code that is distributed across multiple user devices. Performing migrations on a single server that you have direct access to makes migrations a lot easier to test, control, and possibly even revert. Additionaly, multiple client applications may be written in a number of different programming languages so that the respective migration code, which can already be challenging to get correct in the first place, would have to be written multiple times in multiple languages. Lastly, the different clients that implement the [Couch replication protocol](http://docs.couchdb.org/en/master/replication/protocol.html) (like CouchDB, PouchDB, Cloudant, etc.) do not generate the same revision ids (there are plans to unify this behavior, though). We have not talked about revisions and the `_rev` attribute here, but you should know that without consistent `_rev`s creating the same document on multiple clients in parallel could lead to a massive amount of conflicts. This is not to say that running migrations on the client is a bad choice, we just want to caution that it may come with additional baggage.

To round things off, let us suggest a terminology to work with the distinction we have drawn here:

1. **Server-side**: Migrations are performed on a central server-side database and changes or updates are only later replicated to client databases.
2. **Client-side**: Migrations are run directly on client-databases, potentially in parallel, and changes will then be replicated through the system from each client.

We have now established three dimensions of a taxonomy to help us reason about migration strategies. We could even view them as a kind of *orthogonal basis* of the space of possible migrations because we can move along each axis without changing the other. This also means that our migration options multiply: we have identified three storage alternatives, two migration mechanisms, and two places where migrations could be performed for a total of *twelve distinct migration strategies*.

Even though we cannot discuss these options in detail here we have built up a vocabulary for identifying the different approaches. We can, for instance, distinguish the traditional migrations, which are *eager server-side single-version migrations*, from the *lazy client-side single-version migrations* we have named *adapter migrations* in the previous part.

In what follows, we would like to take a better look at one exemplary strategy that happens to be the one we are actually using in practice. As for the rest we found that our taxonomy provides a good entry point for further discussion. Some of the approaches we cannot talk about in this article are still worthy of further scrutiny but we will have to leave it to the reader to follow up with those.




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

##### Versioned API
Instead of leaving old apps high and dry we may very well decide to support them at least for a while. And a common approach to implement multi-version support is through a versioned API. Imagine older clients could get older documents through the `/V1/`-API while newer clients could just talk to the `/V2/`-version. To make this possible we could still keep the CouchDB documents up to the latest version but provide server-side adapters that transform documents to the requested format.

Sounds complicated? Let's take a look at the todo app. Say someone just wrote a `todo-item-1` document to the local database of their web app. But the system has already moved on to supporting `todo-item-2` documents. Luckily, the old app uses the `/V1/`-endpoint when synchronizing `todo-item-1` documents. Behind this endpoint there is an adapter we provided that migrates the document up to the newer version and stores it. The newer clients are save! And when the old wep app wants to get the latest documents it tries to get them, of course, through the `/V1/`-endpoint. Once again, an adapter intercepts the syncing-process and migrates newer documents down to version one. The wep app is save!

As a general migration strategy, this approach looks very promising indeed. It would allow us to write adapters for each API version that could migrate documents on the fly, up and down. However, we cannot pursue this path further at this point because CouchDB does not provide any hooks that we could use to insert our adapters into the data-flow. This would have serious consequences for many aspects of the system including the replication mechanism. Since this is not an option, let's not have this discussion right now and instead focus on what is feasible.


##### learning: ignore unknown attributes

As a side node, we'd like to briefly mention that this approach will only work if old apps do not delete the new `isImportant` attributes. If we only have a simple web app this is of course not a concern because there will be no old apps around. But this will change at some point and we would like to mention the more general lesson here while we're at this point: applications should simply ignore attributes they don't know and persist them along with the data they care about. This way, other applications have the option to use the same schema and only add the attributes they need to provide additional functionality.


## Summary and Evaluation

> The internet is broken - Andi Pieper

TBD

- very complicated and extensive setup
- flexible
- continuous migrations allow for no downtime deploy
- also interesting to apply per version document approach to other dbs (SQL) to minimize downtime and component upgrade
- discuss matrix and why we favor last solution
- outlook
- we will create a proof of concept for migrator
  - already outlined this
  - shit and I already forgot it :/


|                      | Trans&shy;actional Migra&shy;tion | Live Migra&shy;tion | Per Ver&shy;sion Data&shy;bases | Per Ver&shy;sion Docu&shy;ments |
| -------------------- | :-------------------------------: | :-----------------: | :-----------------------------: | :-----------------------------: |
| Drop Version Support | ***                               | -                   | ***                             | **                              |
| Multiple Versions    | -                                 | **                  | **                              | ***                             |
| Legacy App Support   | -                                 | -                   | ***                             | ***                             |
| Distributed Systems  | -                                 | *                   | ***                             | ***                             |
| Seamless Migration   | -                                 | -                   | *                               | ***                             |
| Deduplication        | ***                               | ***                 | -                               | **                              |
| Conflict Safety      | ***                               | ***                 | ***                             | ***                             |
| Purge Version Data   | ***                               | -                   | ***                             | *                               |
| Simplicity           | ***                               | *                   | **                              | ***                             |


- Thanks to all the people who helped here
- Thanks immmr.com
- Follow us on Github, Twitter
- Link to repo
- Ask for feedback, open PR/issue
