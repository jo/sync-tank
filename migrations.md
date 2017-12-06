# [Sync<br/>Tank](/) Distributed Migration Strategies
## How to handle schema changes in CouchDB
{:.no_toc}

## Table of contents
{:.no_toc}

1. this unordered seed list will be replaced by toc as unordered list
{:toc}

## Introduction

> Software development is change management - Ashley Williams, [A Brief History of Modularity](https://www.youtube.com/watch?v=vypCsVm5z28) at JSConf EU 2017


Imagine you've done everything right: you've built this big, offline-first, decentralized, scalable system that supports all sorts of clients. Your agile teams are working on different parts of the software ecosystem. Of course, your application is live and well, customers are happy - and they demand new features.

So you have to change the data structure.

In a classical monolithic server-side architecture this is more or less a [solved problem -> link missing](), not so for decentralized systems. Here we face two formidable challenges:

1. we cannot rely on transactions to transform data, and
2. there might be clients around that still depend on older versions of the schema.

How can we address the problem that there may be clients running older versions of our application? How can we deal with the fact that an older application might be confronted with newer data because other clients in the system have been updated already and are distributing newer versions of that data? These are among the questions we will have to answer if we want to build larger systems with enough flexibility to respond to change-requests.

At the time of this writing, this is still largely unexplored terrain. You will have a hard time finding articles on this topic, let alone guidelines or collections of best practices. In this article, we will make an effort to start filling this gap. In what follows, you will find a detailed exposition of our thoughts on different strategies for managing distributed data migrations in Apache CouchDB. We will start with simple solutions for simple scenarios and work our way up to the complex offline-first, decentralized, multi-client, scalable systems that we are challenged to build.

TBD
- mention that we have a narrower focus: offline capabiliy and CouchDB

<figure>
  <img src="images/authors.jpg" alt="The authors: Matthias and Johannes" />
  <figcaption>The authors: Matthias and Johannes</figcaption>
</figure>

Before you follow us deeper into this discussion and open your minds and hearts to what we have to say, you might want to know who we are and what we do and why you would listen to us in the first place. We are: Johannes J. Schmidt and Matthias Dumke, data architects at a company called [immmr](https://www.immmr.com/), a subsidiary of Deutsche Telekom. The company's main product, [Orbit](https://www.orbit-app.com), is an offline capable app that brings communication to the next level as they say in the product department. It's designed to operate at a large scale in a variety of network conditions (no data available, only GSM, etc.). Our agile teams work on Android, iOS, desktop and web clients as well as on server-side services. Our role is to ensure a seamless integration of data synchronisation between all parts of the system, which is why we are facing on a daily basis all the above mentioned challenges of changing data structures in an offline capable multi client environment.

Moreover Johannes has a decade's worth of experience with distributed databases. He has authored and worked on several widely used tools in the Apache CouchDB ecosystem. He is the main author of the [CouchDB Best Practices](http://ehealthafrica.github.io/couchdb-best-practices/) guidelines he compiled during his work at [eHealth Africa](https://www.ehealthafrica.org/).


## Preliminaries: CouchDB, schemas, and migrations

Going through a detailed technical discussion will be more fruitful if everybody is on the same page from the beginning. This is why in this part we would like to address a handful of general topics including a refresher on CouchDB, our understanding of schemas, migrations, and distributed systems, as well as best practices for working with data schemas in the wild. If you are already familiar with these topics, feel free to skim or skip this section. In the next part we will begin to take a close look at different migration strategies and illustrate them with an example application.

### Who is CouchDB?

As the title of this article suggests we will narrow down the focus of our discussion to software systems backed by CouchDB. In case you didn't know,

> "Apache CouchDB™ lets you access your data where you need it by defining the Couch Replication Protocol that is implemented by a variety of projects and products that span every imaginable computing environment from globally distributed server-clusters, over mobile phones to web browsers."
>
> From the official [Apache CouchDB Documentation](https://couchdb.apache.org/)

We chose to focus on CouchDB because first of all, there is basically no production-ready alternative for a cross-platform data storage that provides synchronization capabilities and is also open source. Check out this [comparison chart](http://offlinefirst.org/sync/) to get an overview of offline capable storage options. And secondly, and pragmatically, we already know a lot about working with CouchDB, so this seems like a sensible place to begin the discussion.

CouchDB is a scalable document oriented store that allows efficient MapReduce data queries via indexing. It is a mature open source project under the Apache foundation that supports clients on all major platforms including web browsers, Android and iOS devices. One of its main features is reliable synchronization of data between different installations via multi master replication (as opposed to master slave replications). We assume some basic familiarity with CouchDB so in the following we will only quickly go over the most important concepts that are relevant in the context of schema migrations. If you need a more in depth introduction to the system, [CouchDB. The definitive Guide](http://guide.couchdb.org/) is a good place to start.

To begin with, let's get some technicalities out of the way: CouchDB provides an HTTP API for all database interactions like storing or retrieving information and managing databases. A single database within the system is just a document that can be created via a simple HTTP request like so: `PUT $COUCH-URL/db-name`. This means that databases are very lightweight and can be created and deleted with ease. As a consequence, and as we shall see below, it is a common practice to create a new database for every user in a system (that's called 'couch-per-user').

Data is transmitted and stored in the form of JSON-documents that can have any valid format you want them to have, plus some attributes such as `_id` and others starting with an underscore that have special meaning. Here's an example document to make this more concrete:

```json
{
  "_id": "todo-item:02cda16f19ed5fe4364f4e6ac400059b",
  "title": "Moonwalk the dog"
}
```

This is a JSON-document storing information about a todo item. We will see more of these documents when we start developing the example application in the next part. But for now we want to focus just on the `_id` attribute, the document id.

CouchDB uses `_id` to uniquely identify documents. If there is any information you need to be unique throughout the database, put it into the `_id`. If you don't set one yourself, CouchDB will create a UUID for you and set it for you, but we recommend to take the opportunity and store more meaningful information here, in other words: make the `_id` semantic! In the example we store the document type along with a UUID. This allows us to quickly identitfy the kind of document we are dealing with if this id is returned e.g. from a query. It also allows us to extract documents by type via the handy [`_all_docs`](http://docs.couchdb.org/en/2.1.1/api/database/bulk-api.html#db-all-docs) query. What's more, it is possible to establish relations between documents through the `_id`. For instance, imagine we also want to store the status of a todo item in it's own document. If we set its `_id` to `todo-item:02cda16f19ed5fe4364f4e6ac400059b:status` we will be able to see at a glance that this is a document of type `todo-item-status` and know which todo item it is associated with. Finding a good strategy for creating `_id`s is one of the challenges when it comes to data design. If you want to learn more about this, please refer to the section [embrace the document id](http://ehealthafrica.github.io/couchdb-best-practices/#embrace-the-document-id) of the CouchDB best practices guide.

As was briefly mentioned above, CouchDB allows to retrieve documents by implementing the powerful [MapReduce](https://static.googleusercontent.com/media/research.google.com/en//archive/mapreduce-osdi04.pdf) pattern. In particular, it allows you to create so-called 'views' that define which data should be returned when they are queried. CouchDB will then build indices to find the requested data quickly and efficiently. As of version 2.0.0 CouchDB also allows queries via [Mango selectors](http://docs.couchdb.org/en/2.1.1/api/database/find.html) that are based on MapReduce and are very performant yet flexible. The ability to query efficiently is of central importance when it comes to data design but there is no need to go into further depth at this point as the discussion of migrations will not require that.

There is one last topic we have to address here if we are going to talk about migrations further down the road, and that is *conflicts*. Conflicts (as opposed to *document update conflicts*) can arise when data is being replicated between multiple instances of CouchDB. To get a feeling for when this may happen, imagine the following 'split brain' scenario: Our friend Andi has CouchDB running on two devices. On both of them he connects to the same database which is currently in the exact same state, meaning all the documents are the same on both devices. This is also true for a document which stores the name of his favorite programming language, Elixir. If there was a network connection between both devices, any data changes on one could be replicated to the other so both would be synchronized. Currently however, there is no network connection. Now Andi goes and changes his favorite language on both devices to something different (in order to break the system, of course, not because his preferences change so rapidly). On the first device, he updates the document to store Lisp, on the other device he updates the same document to store Lua as his favorite Language. Now if both devices reconnect CouchDB will try and bring all data to the same state, but it is inherently unclear which one of the changes should determine the final version of the language document. At this point, CouchDB will keep both versions of the document and leave it to the user to resolve the conflict.

For the time being, these notes will suffice as a review of central CouchDB concepts. We will touch upon many of the points throughout the rest of this article. Next up is an explanation of what we have in mind when we talk about schemas, migrations, and distributed systems.


### Basic concepts: schemas, migrations, and distributed systems

If you want to store and retrieve data in an automated and efficient way, it is important to have some knowledge about the format or structure of this data. This is meta-information specifying things like data attributes and the data types to be used to store them. We would like to think of the **data schema** as all relevant structure-information about the different pieces of data to be stored by an application.

On this general account, a schema formalizes the structure of your data. Many database systems actually require in advance an explicit account of what the data to be stored is going to look like. PostgreSQL for example allows to store relational data where entries have to adhere to one of the [available data formats](https://www.postgresql.org/docs/8.4/static/datatype.html). These formats will even be validated on store-time.

Document databases like CouchDB are sometimes referred to as *schemaless*. This only means that the database will not force you to make your schema explicit before storing data, it does not mean that your data will not have a schema. In the end, a schema is not defined by the database anyways, but by an *application* expecting data to have a certain format. If you don't make your schema explicit, it will simply be *implicit*.

> "Usually when you're talking to a database you want to get some specific pieces of data out of it: I'd like the price, I'd like the quantity, I'd like the customer. As soon as you are doing that what you are doing is setting up an implicit schema."
>
> Martin Fowler, [Introduction to NoSQL](https://www.youtube.com/watch?v=qI_g07C_Q5I)

Thinking in terms of implicit schemas as defined by the application's expectations may require a change of perspective, but it will benefit us in the long run. Moreover, in an upcoming section we will recommend that schemas be made explicit regardless as this will allow us to be more precise about what some piece of data looks like and to keep track of how it changes over time.

Once the application evolves and starts to make new assumptions about the data, flexibility of the (implicit) schema becomes an issue. With a rather lenient document database like CouchDB it may be possible to simply store some additional information in documents at first, but after a while some documents will become outdated and unable to support the functionality new app versions require. At this point we will be forced to transform existing documents into a new form. Changing existing data to adhere to new requirements is called a **data migration**.

The last concept to address are **distributed systems**. According to the Wikipedia,

> "A distributed system is a model in which components located on networked computers communicate and coordinate their actions by passing messages."
>
> Wikipedia, [Distributed computing](https://en.wikipedia.org/wiki/Distributed_computing)

To apply this broad definition to the CouchDB environment, we can interpret database documents as messages, data transfer via replication as message passing, and the various apps on phones and in browers and the backend services as the components located on networked computers. This can mean that data will have to be stored not only on a server but also on user-devices, which in turn must be enabled to synchronize data updates with the rest of the system so all parts can be up to date.

In this distributed scenario, different applications can make very different use of the same data. For instance, an Android app may use data to display a list of todos to the user while a backend service may be interested in the metadata to build usage profiles. We therefore have software that accesses, processes, and stores data in very different locations and according to their very different requirements. For the whole system to be intact it is mandatory that the structure of the data not be changed in any unforseeable way. The different parts of the system, and consequently the different teams working on the different parts of the system, are bound by an (implicit) contract, by the (implicit) data schema they all have to respect. We may even go one step further and say that *the data schema is the effective database API* of the distributed system because it ultimately defines the way in which data can be accessed. From this perspective, a schema-change implies an API change for any part of the system that is dependent on the data. This is why it is of such importance to have a strategy for dealing with schema-changes.

When migrations become necessary in distributed systems, we run into the complex issues this article is about.


### Make your schema explicit!

We have already mentioned that even if people may call your database *schemaless* this does not mean you can actually work without a data schema. Granted, you may be able to change your mind at any time and store differently structured information and the database will be fine with that. This is why schemaless databases have a reputation for facilitating flexible development. But at the very least the data schema will be *implicit* as your applications have certain expectations about what kind information there is in the database. We have found that it is very helpful to be more explicit about the schema. Allow us to argue for that:

1. A data schema that is made explicit makes **communication** about data a lot easier. This is true across time (when you come back to your project three months later) and across teams (when the web team is supposed to add the same functionality to their apps as the Android team).
2. An explicit schema also allows for format checking and **schema validation**, which can expose bugs in applications earlier, and in general lead to more reliable and consistent data. More reliable and consistent data makes application development not only easier, but practically feasible in the first place.
3. Finally, an explicit schema can be **versioned** which makes it possible to track how the format changes over time and even to revert to earlier states.

If this has been convincing to you a plausible follow-up question is how to actually make a data schema explicit. There are a number of tools around to help with that. As this article focuses on CouchDB let us first mention that in the near future the CouchDB team will introduce server-side validations via a *Mango*-like syntax query language. This can be used to enforce requirements on the structure of the data before storing it. You may also know of [JSON schema](http://json-schema.org/), a very flexible schema specification vocabulary. In case you are not familiar with this, here's a simplified example of how we could make the schema of a hypothetical todo item document explicit using JSON schema:

```json
{
  "id": "todo-item-1",

  "properties": {
    "_id": {
      "pattern": "^todo-item:[a-f0-9]{32}$",
      "type": "string"
    },

    "schema": {
      "enum": ["todo-item-1"],
      "type": "string"
    },

    "title": {
      "maxLength": 64,
      "type": "string"
    }

    "isDone": {
      "type": "boolean"
    }
  },

  "required": [
    "_id",
    "schema",
    "title"
  ]
}
```

This specification formulates some expectations and requirements any valid todo item document would have to fulfill. For instance, it requires each such document to set a `title` attribute which must be a string of some maximum length. It is also possible to specify more complex formatting rules the pattern of the `_id` property exemplifies. CouchDB integrates nicely with JSON schema: the above specification could be used to validate documents via the `validate_doc_update` function. To round things off, here is what a corresponding valid document might look like:

```json
{
  "_id": "todo-item:02cda16f19ed5fe4364f4e6ac400059b",
  "schema": "todo-item-1",
  "title": "Clean both swimming pools",
  "isDone": false
}
```

The example features a `schema` attribute on the document with a value of `todo-item-1`. This stores not only the document type but also the schema version of the document, most valuable information once we have to implement schema migrations. We recommend to make the version number a part of the schema id. This way the version is tightly coupled with the data format, while leaving enough room to change document versions independently of each other. For instance, we could upgrade `address-1` documents to `address-2` documents  while the related user data could still be stored in `user-1` documents.

Being explicit about schemas is also the first step to *versioning* them. As a final practical recommendation we suggest to use [semantic versioning](http://semver.org/), properly interpreted to work well with data schemas, in order to keep track of how the schema evolves. We use semantic versioning to determine the version of the *database schema*, which is the collection of all *individual document schemas*. This way it is easy to understand the type and severity of schema changes:

* **Breaking changes** happen every time old documents do not validate against a new schema anymore.
* **Features** are all additions and enhancements that do not affect the validity of existing documents.
* **Fixes** are small improvements like correcting typos in schema descriptions.

```
TBD

data schema versioning through a form of package.json where each type is its own
package, versioned individually, and the dataschema collects them together.

// schema manifest
```

```json
{
  "name": "todo-app",
  "version": "2.1.0",
  "dependencies": {
    "address": "^2.0.0",
    "todo-item": "^1.1.0",
    "user": "^1.3.1"
  }
}
```

```
{
  _id: 'todo-app:todo-item:<uuid>:v:2',
  ...
  app: {
    name: 'todo-app',
    version: '1.2.3'
  },

  schema: {
    'todo-item',
    version: '2.3.4'
  }
}
```

To sum up this discussion, introducing format-requirements on schemaless databases opens up a space between purely implicit schemas and rigidly explicit ones, providing greater flexibility when it comes to specifying and changing schema definitions. This way, we can better find the sweet spot between data consistency and flexibility that we see fit for our projects. Storing the schema version of each document in the document paves the way for implementing schema migrations, which we will now turn our attention to.


## If it ain't broke, don't fix it

There are times when updates to a data schema do not endanger the functionality of a system. In these scenarios, migrations are either straight forward or not even necessary in the first place. We would like to start by looking at some such cases because if you can get away without the multi-client multi-version migration hassle you can save yourself a lot of work. In particular, we will start by looking at unobtrusive additions to the data schema as well as traditional server-side migrations.

### Setting the stage: A toy problem

While this article is not supposed to be a tutorial, it will be instructive to have a working example at hands to illustrate some points. Our example of choice is the 'Hello World' of software applications, the todo app. A simple todo app does not look like the most daunting challenge to face from a data architecture perspective: todos can have titles and texts, maybe a creation date and a done-flag. None of this would make you sit down and write a full article about different strategies for how to accommodate this information. But things get a lot more interesting once we agree to meet a few additional requirements:

1. Users should be able to edit their todos even when their internet connection is unstable or down and the application should synchronize all changes once it reconnects. In other words: the app must be **offline-capable** (what exactly this means will become clearer as we go along).
2. The application development should be **agile**, meaning that we want to support frequent changes to all aspects of the system, including the data schema.
3. Users should be able to use the todo app on **multiple clients**, for instance as a web service and as an iPhone app, with potentially different versions of the software running on different clients.

These requirements will eventually lead us to think about distributed migrations. Of course, this is all way ahead. For now we just want you to know that a tiny todo app can cause a whole lot of trouble further down the road. But as we said this section is about starting simple, so let's start with a simple setup for a simple todo app and take on new migration challenges as we go along.

In the simplest of todo app scenarios, we want to enable a number of users to manage a few, or maybe a few thousand todo items from the comfort of their web browser. The basic building blocks to set up such a service are: a client-side application to provide a nice interface, a server to deliver that application, a central database that stores all those todos, and bit of infrastructure to glue the pieces together. We have decided to use CouchDB as our database and as we mentioned above we chose to set up the system such that every user gets their very own database. This is a common practice in the CouchDB world. It entails that there is no way for one user to get direct access to anybody else's data because they are stored in different databases. Everybody manages their own todos in their own database. And yes: once our product goes viral and there are two million users there will be two million databases. Well, Ops problem. In fact, Ops will be happy to find out that CouchDB comes with clustering abilities so scaling up is not a terrifying prospect (unlike when you run out of space with your relational database).

For now, this piecemeal approach of worrying about one user and one database at a time simplifies our problem. To complete the first step and bring version one of our todo app to the market, all there is to do as far as the schema is concerned is to decide how a single todo item is supposed to look. And since we wanted to start simple, and since the *sine qua non* of a todo item is basically just a title and a flag, here's an example of the first, launch-ready version of a valid todo item document to be stored in a user's CouchDB that also adheres to the schema we specified in the last part:

```json
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "schema": "todo-item-1",
  "title": "reimplement my pet project in Rust",
  "isDone": true
}
```

### The world is changing: new requirements

The first weeks have passed, marketing has done a great job and our app is quite popular, especially with young professional single parents in urban areas. Feature requests are coming in and a decision is made to enhance the product. So we face a new requirement:

```cucumber
As a web app user
I want to mark a todo as important
so that I can find it easier.
```

Obviously the data schema will need some enhancements in order to store that new information. In particular, we will want to add an `isImportant` flag to each todo-item. This change is rather unobtrusive because it leaves already existing todo items intact: since existing documents will not have the `isImportant` attribute, we can simply treat them as not important by default. All we have to do is make sure the app will be able to handle missing `isImportant` flags.

This change was not too hard to implement. A second request that many users have made is the ability to change the color theme of their app:

```cucumber
As a web app user
I want to choose between different color themes
so that I can express myself by personalizing my tools.
```

Obviously, this feature has no implications for the todo items. We decide to introduce a new document type, a `settings`-document, that will store the color information and perhaps other general requirements that will come up in the future. Since we only need one global settings document, we can set the `_id` to something simple.

```json
{
  "_id": "settings",
  "schema": "settings-1",
  "color": "#e20074"
}
```

As with the introduction of an `isImportant` property, the new settings document will not cause existing apps to break. Instead, we can use this document to enhance the experience for users of the new app version. If there is no settings document, the application can simply use the default color that already exists. And if an older app does not know how to deal with settings-documents, it may safely ignore it.

So far we have amended the todo item schema and introduced a new document type. Both operations are non-breaking feature changes to our data schema according to semantic versioning. But now let's look at yet another feature request that will have a deeper impact on our data format:

```cucumber
As a web app user
I want to assign one of many states (`active`, `blocked`, `done`, ...) to a todo item
so that I can have fine-grained control over its progress.
```

We already have the `isDone` property in place that keeps track of an item's progress. This progress information will have to be stored in a different property now, call it `status`. You might be tempted to simply replace one field with another, which would lead to documents like the following:

```json
{
  "_id": "todo-item:ce2f71ee7db9d6decfe459ca9d000df5",
  "schema": "todo-item-2",
  "title": "change the world",
  "isImportant": true,
  "status": "active"
}
```

This is a valid strategy, but we can do better. Up to this point, a todo item has been a rather coherent set of attributes. Yes, a title might have changed, people were able to mark todos as important and at some point in time many of the items would have been marked as done. In all those cases the document would have had to be updated. However, these changes can be expected to be relatively infrequent. In contrast, we're now introducing the concept of a multi-valued status that may change many times across the lifetime of a todo item. It would be great if we didn't have to update the whole document every time someone changes the status. This is why we propose to split the document in two, have a todo item document and save the status separately in a new document type. With these changes, the remaining core todo item might look like this:

```json
{
  "_id": "todo-item:ce2f71ee7db9d6decfe459ca9d000df5",
  "schema": "todo-item-2",
  "title": "change the world",
  "isImportant": true
}
```

And this might be the corresponding status document:

```json
{
  "_id": "todo-item:ce2f71ee7db9d6decfe459ca9d000df5:status",
  "schema": "todo-item-status-1",
  "status": "active"
}
```

Note how the association of status and todo item is established via the `_id` attribute that is not just used to determine the document type. Splitting the document now enables us to group attributes that commonly change together. Here is not the place to get into a detailed discussion of data design, but feel free to take a look at the [couchdb-best-practices](http://ehealthafrica.github.io/couchdb-best-practices/) guidelines if this discussion got you interested.

At this point we have some idea for how to change the data schema in order to react to the latest feature request. We would like to release a new version of the web app that allows users to set different progress states for their items. But wait! What about the items that have been created in the past? The previous approach does not seem to work anymore: we cannot simply choose some sensible defaults and assume that old documents are still valid. Instead we have to find an explicit way to map the old `isDone` values to the new `status`, in other words: we need a data migration!


### Traditional server-side migration

We are now at a point where a new version of an application may be confronted with older documents in the system that it does not know how to handle. But this prospect does not scare us because there is a common practice that is used in this scenario, and one that you probably know well already. We will call this the *traditional server-side migration*.

<figure class="diagram" id="figure-1">
  <img src="images/transactional-migration.svg" alt="Schematic view of traditional server-side migration" />
  <figcaption>
    <b>Figure 1: Traditional Server-Side Migration.</b>
    <span>
      Introducing a new version of an application may require us to update existing documents (white) to a new version (green). This switch happens at a single point in time. Some documents (two-colored example) can be handled by both app versions and do not need to be transformed at all.
    </span>
  </figcaption>
</figure>

Since we have stored all our data in one central database, it will be easy enough for us to access all existing todo items and update them to adhere to the new schema. Ruby on Rails's way of handling migrations provides a very straight forward example of this approach. In Rails we would define a migration that formalizes the schema change (prepare the database to store the new `status`, move existing `isDone` information over into the `status`, remove the `isDone` field from the todo item table). We would then take the system down, run the migration (the famous `rails db:migrate`, formerly `rake db:migrate`) and hand out the updated application once the database is back up. If anything goes wrong during this process, there will be a rollback because the migration is wrapped into a transaction. During the process we will of course incur some downtime, but on the plus side we always have consistent and up to date documents and everyone will get the latest version of our application.

This same strategy works with CouchDB as well. Let us roughly sketch out the main steps to go through when performing a **traditional server-side migration on CouchDB**. Here's our recipe: First you need to setup a cluster in case you want to revert or abort the migration at some point. Then switch to maintenance mode. Perform a replication of all data from the old to the new cluster. Now run a migration script to update data on the new cluster. If a check verifies that migration was successful, switch to the new cluster and switch off maintenance mode. Done.

[Figure 1](#figure-1) illustrates the traditional migration strategy. It shows how both the application and the documents are updated together in a single step. White documents can be handled by the white version of the app while the green app needs green documents. A special case is the two-colored document. There might be documents that do not need to change during a migration and that can be handled by multiple versions of the client. Think of the settings document we have added previously. Even if we have to change the structure of todo items this does not mean we have to change how the settings document looks.

This migration procedure is very common for the type of monolithic centralized setup we have described so far. It does not come without some [problems of it's own](https://about.futurelearn.com/blog/your-database-is-a-distributed-system) but overall it is a well established and reliable practice. Alas, this is not a viable solution anymore once we bring in some of the more challenging requirements we have so far ignored to keep our setup simple.


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

When a user can create or edit todo items even if the client is offline we need to provide a way to synchronize any changes once it comes back online. Luckily we have CouchDB on our team! There are a number of client-side adaptations like [PouchDB](https://pouchdb.com/) for browsers or [Cloudant sync](https://www.ibm.com/analytics/us/en/technology/offline-first/) for phones that provide CouchDB-like storing capabilities for clients and implement the Couch replication protocol so synchronizing data between different parts of the system becomes simple and fun.

CouchDB is great when it comes to building offline capable apps and managing data synchronization. But clients that are offline may miss a software update. When this happens, there may be outdated clients that are still reading and writing data that adheres to older schemas. This problem becomes even more obvious once we build native apps that may have to be updated manually. So why not introduce them appropriately:

```cucumber
As a customer
I want to use the todo app on my iPhone and through a web interface
so that I can fit my workflow to my life and not the other way round.
```

If we provide users not only with a web app but also with native clients then updating the data schema to a newer version comes with certain additional challenges. Imagine someone is using our todo app both through the web interface and on their phone. When we release a new version of the software that includes a breaking change to the data schema the web app gets updated immediately once the page is reloaded. But the app on the phone may have to be updated manually, and this could happen now, or soon, or someday, or - never. In this situation we will have multiple clients with multiple versions in the system.

In this more complex scenario we are now confronted with three formidable challenges: we want to support multiple clients running multiple app versions, we want them not to break when the network connection is unstable, and we want to develop our software incrementally and stay agile, pivoting and releasing new versions in response to user demand. How could this ever work, you wonder? Let's wonder together.

### Traditional migrations fail

Why can't we just reuse the traditional migration strategy that has worked well before and update all existing documents on the server? This would mean that we picked a single point in time where we updated all documents on the server to be compatible with the new data schema. The changes would then be replicated to the clients' local databases.

This approach fails for a couple of reasons. Let's first consider the case where a web app may be offline thanks to our latest enhancements. Here's a chain of events that breaks the system: Haneen is using the web app, version one, in her browser right now. Because she is currently on the train she is editing todos in offline mode. She has just created a todo to pluck some flowers for her friend's birthday. Meanwhile we release a new version of the app and migrate all existing data on the server-side database. Back at home, Haneen's app synchronizes the recent changes. The updated documents get replicated from the server-side CouchDB to her in-browser database. This may take a moment but we can assume we find a way to verify the update is complete before allowing the app to resume. Eventually the web app, which has all the latest features now, is ready to work. But what about the flowers-todo? It was created offline according to the previous version of the data schema. It still has the simple `isDone` flag instead of the new fancy `status`. The new version of the app does not know how to deal with this old document. It might even break. Utter disaster.

This problem on its own can be dealt with, and the solution has two parts. First we need to make sure that the new app does not break when it encounters older documents. This should not be too hard. It basically has to ignore the `isDone`-flag in this case. And the fact that a `status`-document is missing should cause no troubles for the application anyways because this could also happen under normal circumstances where documents simply need some time to be synchronized. The second part of the solution is a backend-service that continuously updates older documents that might be coming in. In our example, the old flower-todo will be replicated to the server at some point. The service, which would look at incoming documents, would notice that the version is outdated and perform a *post hoc* migration. Then the new flower-todo is synced back to the clients. Problem solved.

As this discussion shows, there are times when you can get away with a traditional server-side migration by amending it with a continuous update of potentially outdated documents. If you just want to provide an offline capable web app, this strategy may be just what you need. Of course, our scenario is more complex because we want to provide native clients as well. This comes with additional difficulties.

If there are clients with multiple versions operating on the same data, the traditional migration strategy is lost beyond reclaim. To see why it fails so badly, consider another exemplary scenario: Basti is using the app, version one, as a web app and he also has it installed on his iPhone. Today is migration day so you update the documents on the server-side database and prepare to hand out the new version of the app. When Basti reloads the web site, he will get the new version of the app right away. But what about the iPhone app? Of course, all the updated data will be replicated to the iPhone's local database as well, but if Basti did not make sure to update the app immediately, it starts to show some weird behavior or simply breaks because we effectively deleted the old documents it was depending upon. Utter disaster.

Is it possible that this problem arises because we perform the data migration centralized on the server-side database? What if we did it directly on the clients instead? The setup could look like this: once a client updates to a new version, it pauses for a moment and runs the traditional migration on its local database. When this is finished, it resumes operation and now works with the new data. This will fail as well, however, and for a very similar reason. Remember that we are dealing with a distributed system that communicates through CouchDB's replication mechanism. In the current scenario, this would entail that the document updates that have just happened on a single client will be distributed throughout the system and will eventually update and overwrite documents on older clients as well. Basti's web app just broke his iPhone app.

The traditional migration approach, server-side or client-side, with or without a continuous update amendment, cannot be successful in our complex scenario. There is no easy way to say this: we have to get creative and look for alternative solutions.

### Searching for alternatives

When we introduced the `isImportant` flag to our data schema we mentioned only passingly that old documents without this flag would still be valid. This was because an app could treat a missing flag as `false` and the todo item as not important by default. We didn't need a big migration for this change at all because by interpreting the existing data appropriately the *clients* were able to ensure the smooth introduction of a new feature. Perhaps we can take cues from this pleasant experience and have the applications themselves play a bigger role in data migrations.

<figure class="diagram" id="figure-2">
  <img src="images/live-migration.svg" alt="Schematic view of adapter migration" />
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
  "schema": "todo-item-1",
  "title": "Calculate the carbon footprint of a bitcoin transaction",
  "isDone": true
}
```

And it returns the updated documents, a todo item and a status-document with a plausible mapping from `isDone` to `status`:

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818",
  "schema": "todo-item-2",
  "title": "Calculate the carbon footprint of a bitcoin transaction"
}
```

```json
{
  "_id": "todo-item:8f5e6edb6f5208abc14d9f49f4003818:status",
  "schema": "todo-item-status-1",
  "status": "done"
}
```

From this point on the app knows how to proceed. Using the adapter, it can treat older documents just as if they were new ones.

Adapters have their own issues, though. Who would have thought! First of all, if we update documents once we *read* them, we might encounter massive amounts of duplicates in the system. This is because there can be multiple clients that read documents at the same time, e.g. when they display lists of todos on different devices. But if new versions of documents are created in several places at the same time this creates conflicts when the updates get replicated across the system.

To avoid those conflicts old documents that are read through adapters should only be changed in the database when their content changes, in other words: the actual migration should happen *on write*. But even then we could get into trouble if an updated client starts migrating documents while there is still an older client that depends on the older documents. We seem to have gotten nowhere so far, only that we now have to maintain a potentially large number of adapters between who knows how many different versions of the data schema that work on different clients, so they will probably have to be written in different languages.

It seems like the basic problem with all approaches so far is that we try to maintain *multiple* client versions that are all supposed to work with a *single* version of the data schema. And how could this work in the first place?

So is there a way to maintain multiple versions of the data schema in parallel? CouchDB will certainly not object if the same information is stored in different formats - as long as the `_id` attributes of the respective documents are unique. Maybe there is a way to run *non-destructive migrations* then. For instance, adapters could read old documents and later store the information in a new document with an appropriate `_id` without overwriting the existing documents. This way, old apps won't break immediately. But if we don't make sure old documents reflect the latest changes from user interactions, how can old apps still be up to date when they are using old documents? And can they even deal with the new documents that can arrive from newer clients? Or is there a way to ignore those? And what if managing the `_id` gets too messy? We said databases are lightweight and cheap, can't we just have a completely new database for every version of the data schema?

Wow, this investigation has gotten out of hands! It looks like we need a more structured approach to get a grip on this messy collection of problems. But rest assured that our efforts will pay off soon. We've seen some strategies fail, but we also understand better what we are up against. Plus we have collected a number of ideas and built a few tools in the process. Let's switch gears now and approach our discussion with a tad of rigorous discipline.


## A taxonomy for reasoning about migrations

The common approach when it comes to schma migrations is to modify data that currently adheres to one format so that it matches new format requirements. This process is usually destructive: the old data is assumed to be obsolete after the migration and is eliminated during the migration process. But we are now dealing with a situation where multiple schema versions may have to exist in parallel because there can be multiple clients in a system that run different versions of some application software.

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

We have also already encountered the second migration mechanism in the form of adapters that were briefly discussed in the last section. In contrast to migration tasks, adapters perform migrations *on-the-fly* as it were. It is characteristic for this approach that document updates happen only when documents are actually needed by an application. Data that is never touch will never be migrated. To get a handle on this *how-to*-dimension we propose to use the following terminology:

1. **Eager**: Migration strategies that perform document updates directly for all available data.
2. **Lazy**: Strategies that only update documents when documents are actually needed.

A major difference between the two approaches is that lazy migrations have to intercept the read (and possible the write) process of the applications while eager migrations can happend independently of any application behavior.


### Where is data being modified?

As a third and last dimension of a useful taxonomy we suggest to take into account whether migrations are performed on the server-side database or on the client. Traditional migrations are only run on the server, but we have already started to consider alternatives in the previous discussions.

Running migrations on client-side databases comes with additional difficulties. For instance, it will be harder to fix bugs and maintain code that is distributed across multiple user devices. Performing migrations on a single server that you have direct access to makes migrations a lot easier to test, control, and possibly even revert. Additionaly, multiple client applications may be written in a number of different programming languages so that the respective migration code, which can already be challenging to get correct in the first place, would have to be written multiple times in multiple languages. Lastly, the different clients that implement the Couch replication protocol (like CouchDB, PouchDB, Cloudant, etc.) do not generate the same revision numbers. We have not talked about revisions and the `_rev` attribute here, but you should know that without consistent `_rev`s creating the same document on multiple clients in parallel could lead to a massive amount of conflicts. This is not to say that running migrations on the client is a bad choice, we just want to caution that it may come with additional baggage.

To round things off, let us suggest a terminology to work with the distinction we have drawn here:

1. **Server-side**: Migrations are performed on a central server-side database and changes or updates are only later replicated to client databases.
2. **Client-side**: Migrations are run directly on client-databases, potentially in parallel, and changes will then be replicated through the system from each client.

We have now established three dimensions of a taxonomy to help us reason about migration strategies. We could even view them as a kind of *orthogonal basis* of the space of possible migrations because we can move along each axis without changing the other. This also means that our migration options multiply: we have identified three storage alternatives, two migration mechanisms, and two places where migrations could be performed for a total of *twelve distinct migration strategies*.

Even though we cannot discuss these options in detail here we have built up a vocabulary for identifying the different approaches. We can, for instance, distinguish the traditional migrations, which are *eager server-side single-version migrations*, from the *lazy client-side single-version migrations* we have named *adapter migrations* in the previous part.

In what follows, we would like to take a better look at one exemplary strategy that happens to be the one we are actually using in practice. As for the rest we found that our taxonomy provides a good entry point for further discussion. Some of the approaches we cannot talk about in this article are still worthy of further scrutiny but we will have to leave it to the reader to follow up with those.


## Chesterfield Migration

We have come far. We now share a common vocabulary and a systematic understanding of distributed migration strategies, we have seen when they are necessary and when you might get away without them. In this section we are going to take a good look at one particular strategy that can work well in practice. We will not shy away from addressing a range of problems and difficulties that emerge when this approach is implemented on top of CouchDB and we will propose a set of solutions for them, so be prepared for a more detailed technical discussion.

The strategy we are going to present is not the only reasonable choice as should be clear from the previous discussion. Still we believe that among the options we discussed it allows for a clean and maintainable implementation given that we want to support the complex scenario we have built up in the previous sections. To recap: we demand that our system support offline-capable applications and clients on multiple platforms that require different schema versions of application data, all the while providing full backwards-compatibility for older apps or services and enabling agile development.

<figure>
  <img src="images/chesterfield.jpg" alt="The authors: Matthias and Johannes" />
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
  <img src="images/offline-camp-migration-session.jpg" alt="Migration session at Offline Camp 2017 Berlin" />
  <figcaption>Foto by Gregor Martinus: Migration session at Offline Camp 2017 Berlin</figcaption>
</figure>

The [offline camp berlin 2017](http://offlinefirst.org/camp/berlin/) provided an excellent opportunity to discuss the strategy with members from the offline-first community and we gratefully appreciate the thoughtful comments from Gregor Martinus, Bradley Holt, Martin Stadler (a former immmr-colleage!) and others. These discussions gave us a lot more confidence that we have found a robust approach that can persist through a number of challenging edge-cases.

### An eager server-side multi-version migration strategy

The chesterfield migration is an *eager server-side multi-version migration*. Our implementation of this features, in broad strokes, a micro-service listening to CouchDB's `_changes`-endpoint for document updates and activating different *transformers* on demand which perform the actual document migration, all of which is happening on the server-side database with changes being replicated to client-databases afterwards. [Figure 3](#figure-3) illustrates the idea: when necessary, transformers create multiple versions of documents so that a single shared database can support multiple app versions.

<figure class="diagram" id="figure-3">
  <img src="images/per-version-docs.svg" alt="Schematic view of chesterfield migration" />
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
so that I can be better organized even when there are a lot of things to do.
```

To keep things simple, we will require each todo item to belong to exactly one group. This association can be established via an additional `group`-attribute that stores the respective group name for each todo item. If no particular group is chosen by the user, items should be assigned to a group named "default". You could argue that this didn't have to be a *breaking* schema change, that it would be possible for apps to handle older items without `group`-attributes as though they were *default* items. But let's assume for the sake of argument that in our case this is not good enough for app developers. They *need* every todo item to state its group. And as we argued that app expectations define a data schema to begin with, so they also define what counts as a breaking change. To make a long story short: we need to *require* a `group`-attribute for each todo item, and we need to go over all existing items and add them to the default group. So we indeed need to run another migration.

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

These are the three major schema versions that we have to maintain in parallel. In order to keep all versions up to date when one piece of data changes it will be necessary to update existing documents in multiple versions. For instance, if a user creates a todo item with their very old app that still runs version one, we need to migrate documents up so that the new todo is not lost on devices that run newer versions. Similarly, we need to migrate changes down to support older apps. Apart from the actual migration logic this setup requires some additional infrastructure to work well. First, we will need to distribute documents with multiple versions throughout the system, and then we will need to manage documents with multiple versions from inside the applications. Let's take a closer look.

#### Replication channels

Chesterfield is a *server-side* migration. This means that apps will replicate documents from their local databases to the server-side database where *transformers* will produce all relevant versions of the documents that will then be synchronized across the system. If we dig a little further though, we will find that not *all* versions will have to be propagated to *all* clients. For instance, if a new client produces a todo item according to version three, and if the server migrates this document down to versions one and two, the newer client would not be interested in receiving the older documents. We can save a lot of traffic if we can prevent newer clients from receiving older documents.

Do we also need to prevent clients from receiving *newer* document versions than they can currently handle? The answer depends on which side we favor in a trade-off between replication traffic and ease of application update. On the one hand, preventing newer documents from being replicated to clients that are not yet ready results in lower traffic. On the other hand, the newer documents will have to be replicated anyway once the application is updated and is expecting newer schema versions.

And there is more: if we allow newer documents to reach clients that are not yet ready we can support what we would like to call **seamless migrations**, migrations without downtime. This is possible because we could migrate documents up to a new version, distribute them across the system, and only once the data had been propagated would we hand out newer client versions. Once the apps get updated, they find all the new data they need is already in their local database!

Our short discussion has elicited two new requirements with respect to replication management:

1. Do not replicate older documents to newer clients. They don't need those anymore.
2. Replicate newer documents to older clients. They can use those once they get updated.

Replication flow management is realised in CouchDB through [filtered replications](http://docs.couchdb.org/en/2.1.1/replication/protocol.html). The traditional way to specify which data to replicate is through filter functions. However, as of CouchDB 2.0.0 there is a new and very performant way to do implement filtered replications with the help of [Mango selectors](http://docs.couchdb.org/en/2.1.1/api/database/find.html#find-selectors). Let's see them in action.

The following listing shows a selector that would be used by todo apps of version two to replicate documents from the server-side database. Note how the selector allows for specifying boolean operations on document attributes via `$or`, `$and`, and `$nin` as well as magnitude comparisons via `$gte`.

```json
{
  "selector": {
    "$or": [
      {
        "$and": [
          { "schema": "todo-item" },
          { "version": { "$gte": 2 } }
        ]
      }, {
        "$and": [
          { "schema": "todo-item-status" },
          { "version": { "$gte": 1 } }
        ]
      }, {
        "$and": [
          { "schema": "settings" },
          { "version": { "$gte": 1 } }
        ]
      }, {
        "schema": {
          "$nin": ["todo-item", "todo-item-status", "settings"]
        }
      }
    ]
  }
```

There are a few noteworthy aspects about this selector. We said it's supposed to be used by clients that work with schema version two. Recall from the example manifests that schema version two requires todo items of version two. But since we want apps to receive all newer documents as well we ask for replication of todo items of version two *or higher*. Similarly, schema version two works with status and settings documents of version one and so we replicate documents of version one *or higher*.

These points are relatively straight forward from the previous discussion. But what about the last specification that asks for replication of all document types *except* for todo items, status, and settings documents? Again, the rationale behind this is that we want to client to be open to new schema versions. Thus including all currently unknown versions in the replication, new document types that may be added in the future would be replicated as well so as to allow for scheamless migrations.

As a last point we like to reiterate that replications take time and may lead to temporarily incomplete data. For instance, the todo item may have already been replicated while the corresponding status document is still missing. But this is a general learning about CouchDB: clients have to be able to deal with this kind of incomplete data anyway. If you really need pieces of data to be present together consider keeping them together in one document.

#### Multiple schema versions in a single database

```
- managing multiple versions in a single database
  - references without versions so associations remain intact
  - clients restrict access to versions they know
  - querying data: views and mango selectors
    -> version in _id
  - optional throw-away of old data via local filtered replication on tmp-db
  - no need to change urls for client
```




### Transformer modules

```
- on server side, migration is implemented as follows:
  1. listen to changes, filter by old (source) schema version
  2. apply migration to each doc
  3. create doc with new version or update
- works in both directions
- complexity analysis (?)
- server-side transformers consisting of
  - type registry
  - transformation engine
  - update handler
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

This is probably the biggest concern with live migrations but also the hardest to work through so we will discuss this last. It means that an old app has to be shut down until it is updated. For a simple webapp this may be as easy as requiring a page reload, for a desktop app it may require a restart, but for iOS or Android apps it may require users to visit some appstore and go through a whole procedure for upgrading an application. And what if product decides that updates should be paid for by device? Do we shut down all older apps once someone upgrades a single device?

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

