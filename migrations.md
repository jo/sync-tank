# Distributed Migration Strategies

## Table of contents
1. Introduction

2. Setting the stage: A toy example

3. Concepts: Schema, Migrations, and Distributed Systems

4. Server-side Todo-app

5. Non-breaking feature
  - New requirement: important-flag
  - Schema: important (boolean)
  - caveat: app must handle missing attributes

5. Transactional Migration
  - New requirement: enhance done to progress
  - Schema: status (string)
  - transactional (describe migration procedure)
  - Reference rails db:migrate

6. Client-side migrations
  - New requirement: webapp with offline-support
  - scenario: couch per client, one device per user
  - procedure (describe migrate procedure, no migration on the server!)
  - caveat: single-client only (leads to conflicts)

7. Live Migration
  - New requirement: multi-client support
  - scenario: multiple devices per user, connect with one server-db
  - Schema: extract status to extra document
  - (similar concerns addressed in best practices)
  - describe strategy: write adapters: read old versions, write current version
  - benefit: data-efficient because updates only happen when necessary
  - caveat: code complexity / adapter abundance vs expensive reads - O(exp)?
  - caveat: force update of app because old versions cannot be supported
  - caveat: impossible to drop legacy-app-support or to purge old documents

8. Per-version-database
  - New requirement: Legacy support
  - scenario: apps for Android and iOS with update-hurdles
  - strategy: create a database per version with bi-directional server-side migrations
  - benefits: single responsibility (easier to maintain and test)
  - caveat: duplicates build up on the server + lot of data-movement on client
  - caveat: manage many dbs on the server
  - caveat: not seamless because upgrade takes time

9. Per-version-documents
  - more elegant strategy: keep multiple document-versions in the same database
  - review and repeat context
  - description

10. Summary and Evaluation
  - discuss matrix and why we favor last solution
  - outlook


## 1 Introduction

Imagine you've done everything right: you've built this big, offline-first, decentralized, scalable system that supports all sorts of clients. Your agile teams are working on different parts of the software ecosystem. The database has become the glue that binds the teams together, the data structure functions as a kind of contract that all teams have agreed on. Of course, your application is live and well, customers are happy - and they demand new features.

So you have to change the data structure.

In a classical monolithic server-side architecture this is more or less a solved problem [cf...], not so for decentralized systems. Here we face two formidable challenges:

1. we cannot rely on transactions to transform data, and
2. there might be clients around that still depend on older versions of the schema.

How can we make sure different API-versions are still supported in our distributed system? How can we address the problem that there may be clients running older versions of our application after being offline for some time? How can we deal with the fact that an older application might be confronted with newer schema versions because other clients in the system have been updated already and are distributing data that adheres to these newer versions? These are among the questions we will have to answer if we want to build larger systems with enough flexibility to respond to change-requests.

At the time of this writing, this is still largely unexplored terrain. You will have a hard time finding articles on this topic, let alone guidelines or collections of best practices. In this article, we will make an effort to start filling this gap. In what follows, you will find a detailed exposition of our thoughts on different strategies for managing distributed data migrations. We will start with simple solutions for simple scenarios and work our way up to the complex offline-first, decentralized, multi-client, scalable systems that we are challenged to build.

Before you follow us deeper into this discussion and open your minds and hearts to what we have to say, you might want to know who we are and what we do and why you would listen to us in the first place. We are: Johannes J. Schmidt and Matthias Dumke, data architects at a company called [immmr](https://www.immmr.com/), a subsidiary of Deutsche Telekom. The company's main product is an offline capable app that brings communication to the next level as they say in the product department. It's designed to operate at a large scale in a distributed scenario. Our agile teams work on Android, iOS, desktop and web clients as well as on server side services. Our role is to ensure a seamless integration of data synchronisation between all clients, which is why we are facing on a daily basis all the above mentioned challenges of changing data structures in an offline capable multi client environment.

Moreover Johannes has a decade's worth of experience with distributed databases. He has authored and worked on several widely used tools in the Apache CouchDB ecosystem. He is the main author of the [CouchDB Best Practices](http://ehealthafrica.github.io/couchdb-best-practices/) guidelines he compiled during his work at [eHealth Africa](https://www.ehealthafrica.org/).


## 2 Setting the stage: A toy problem

While this article is not supposed to be a tutorial, it will be instructive to have a working example at hands to illustrate some points. Our example of choice is the 'Hello World' of web applications, the todo-app, though what we have to say is not only relevant for web applications. A simple todo-app does not look like the most daunting challenge to face from a data architecture perspective: todos can have titles and texts, maybe a creation date and a done-flag. None of this would make you sit down and write an article about different strategies for how to accomodate this information. But things get a lot more interesting once we agree to meet a few additional requirements:

1. Users should be able to edit their todos even when their internet connection is unstable or down and the application should syncronize all changes once it re-connects. In other words: the app must be offline-capable (what exactly this means will become clearer as we go along).
2. The application development should be agile, meaning that we want to support frequent changes to all aspects of the system, including the data schema.

These two requirements lead us to think about distributed migrations. Because imagine product comes up the next day with their latest idea: a todo should no longer just be open or done but should hold a number of different progress states like 'started', 'blocked', 'rejected', 'completed', etc. No problem, right? We just tweak the schema a bit, the todo state is no longer a Boolean but some form of String or Enum that can hold multiple values and we are done. - But wait, what about the todos that are already in the system? - Maybe we can update them? Or maybe the apps should be able to deal with both kinds of todos? - So we should support two schema versions now and old documents are still valid? And what if someone is using the old version of the app on their phone? The old version doesn't know anything about the new progress states. How could it possibly deal with documents of the new format? - We could force people to upgrade their apps so everybody at least agrees on the new schema. - So you want to make the entire application unusable unless someone goes to the appstore and updates to the latest version? This is not very appealing. - Do you have a better idea?

In fact, we do. Stay tuned.

At this point we have gotten a bit ahead of ourselves and are already discussing migration strategies, albeit in a somewhat hasty and unstructured manner. But at least we understand the problem now.

This is actually a good time to take a step back and clarify some concepts that will be important throughout this whole discussion. Before we talk about schema migrations in distributed systems, let's talk about what schemas and migrations and distributed systems are in the first place.


## 3 Basic concepts: schemas, migrations, and distributed systems

["Software development is change management" - Ashley Williams]

If you want to store and retrieve data in an automated and efficient way, it is important to have some knowledge about the format or structure of this data. This is meta-information specifying things like data attributes and the data types to be used to store them. We would like to think of the **data schema** as all relevant structure-information about the different pieces of data to be stored by an application.

On this general account, a schema formalizes the structure of your data. Many database systems actually require in advance an explicit account of what the data to be stored is going to look like. PostgreSQL for example allows to store relational data where entries have to adhere to one of the [available data formats](https://www.postgresql.org/docs/8.4/static/datatype.html). These formats will even be validated on store-time.

But even if you opt for a schemaless document database like [MongoDB](https://www.mongodb.com/) or [Apache CouchDB](https://couchdb.apache.org/) - which we will choose in this article to power our example application - you will not get away from data schemas at all. This is because in the end what defines a schema are the assumptions the applications make.

> "Usually when you're talking to a database you want to get some specific pieces of data out of it: I'd like the price, I'd like the quantity, I'd like the customer. As soon as you are doing that what you are doing is setting up an implicit schema."
>
> Martin Fowler

Thinking in terms of implicit schemas as defined by the application's expectations may require a change of perspective, but it will benefit us in the long run. Having said that, we would still like to mention that there are tools for making schemas more explicit even when working with schemaless databases. There is, for example, the very flexible [JSON Schema](http://json-schema.org/) specification, and CouchDB is going to introduce server-side validations via the Mango-Query language in the near future. This allows us to harden the data schema, enforcing requirements on the structure of the data before storing it, to the degree we see fit. We might, for example, require the titles of our Todo-items to be strings of a certain maximum length, while we want to pose stricter format-requirements on the ids of our documents, which should be, say, strings that have to match a very specific pattern. To give but one example of how this might look in a JSON-schema document, consider the following abbreviated specification:

```js
// using JSON-schema to formalize format-requirements of our Todo-documents...

{
  "id": "todo-item",

  "properties": {
    "_id": {
      "pattern": "^todo-item:[-a-f0-9]{36}$",
      "type": "string"
    },

    "title": {
      "maxLength": 120,
      "type": "string"
    }

    "isDone": {
      "type": "boolean"
    }
  },

  "required": [
    "title",
    "isDone"
  ]
}

// ...and a corresponding valid document

{
  "_id": "todo-item:00000000-00000000-00000000-00000001",
  "title": "Clean the dishes",
  "isDone": false
}
```

In short, introducing format-requirements on schemaless databases opens up a space between purely implicit schemas and explicit ones, providing greater flexibility when it comes to specifying and changing schema definitions.

Flexibility of the (implicit) data schema becomes important once the application starts to change. Let's revisit the example from above: our todo-app is suddenly required to deal not just with a done-state that can be true of false, but is supposed to display one of many progress-states. If we decide to completely remove the done-state and replace it with a progress-state, we will have to change all the existing todos when we deploy the new version of our application. Changing existing data to adhere to new requirements is called a **data migration**.

In the current example, we are not *forced* to migrate existing data. It would be possible to get around a migration if we kept the done-state, introduced a new progess-state, and gave our application the ability to deal with both versions of todo-documents. Of course, this strategy has its own problems, as we will discuss in greater detail below. For now, we would just like to settle on an account of what data migrations are: they are changes to the structure and format of existing data to meet new requirements.

The last concept to address are **distributed systems**. According to the Wikipedia,

> "A distributed system is a model in which components located on networked computers communicate and coordinate their actions by passing messages."
>
> Wikipedia

This is a broad definition, which can be made more specific to fit the focus of our current discussion and our running example. In what follows, we are particularly interested in systems with distributed data storage. In our example we look at a distributed todo application with offline capable clients, which implies that we will have to store data not only on a server but also on the respective user-devices, which in turn must be enabled to synchronize data updates with the rest of the system so all parts can be up to date.

In this distributed scenario, different applications can make very different use of the same data. An Android App may use the data to display a list of todos to the user while a backend service may be interested in the metadata to build usage profiles. We therefore have software that accesses, processes, and stores data in very different locations and according to their very different requirements. For the whole system to be intact it is mandatory that the structure of the data is not changed in any unforseeable way. The different parts of the system, and consequently the different teams working on the different parts of the system, are bound by an (implicit) contract, by the (implicit) data schema they all have to respect. We may even go one step further and say that *the data schema is the effective database API* of the distributed system because it ultimately defines the way in which data can be accessed. From this perspective, a schema-change implies and API change for any part of the system that is dependent on the data. This is why it is of such importance to have a strategy for dealing with schema-changes.

When migrations become necessary in distributed systems, we run into the complex issues we have already briefly encoutered above. But we're not going to address all of them at once. Instead, let's shift gears and start building our todo-application again, this time from scratch and with only some simple requirements at first.


## 4 A Server-side Todo-app

In the simplest of todo-app scenarios, we want to enable a number of users to manage a few, or maybe a few thousand todo items from the comfort of their web browser. The basic building blocks to set up such a service are: a client-side application to provide a nice interface, a server to deliver that application, and a central database that stores all those todos, and bit of infrastructure to glue the pieces together. We can all agree on that. Of course, we will most likely not agree on the technology stack to actually build this, including which database system to make use of. Let us sketch out our approach.

You might be tempted to go for a big relational database where you store user data and todo items and implement has-many relations between them via foreign keys. We've all done that at some point. But here we anticipate that we will need a lot of flexibility in the future and that this big, centralized RDBMS might not be malleable enough to adapt to our upcoming needs, especially when it comes to offline-capable client applications at some point. This is why we advocate a different choice at this point and begin developing our todo-app using Apache CouchDB. In case you didn't know,

> "Apache CouchDB™ lets you access your data where you need it by defining the Couch Replication Protocol that is implemented by a variety of projects and products that span every imaginable computing environment from globally distributed server-clusters, over mobile phones to web browsers."
>
> From the official Apache CouchDB Documentation

That sounds promising, doesn't it? Now the way we set up the system is according to a common practice in the CouchDB world: each user gets their own database on the server and can connect to it through a client application. This means there is no way to get access to anybody else's data and we don't even have to worry about things like associations and foreign keys for the time being. Everybody manages their own todos in in their own database. And yes: once our product goes viral and there are two million users there will be two million databases. Well, Ops problem. In fact, Ops will be happy to find out that CouchDB comes with clustering abilities so scaling up is not a terrifying prospect (unlike when you run out of space with your relational database).

For now, this piecemeal approach of worrying about one user and one database at a time simplifies our problem. To complete the first step and bring version one of our todo-app to the market, all there is to do as far as the schema is concerned is to decide how a single todo item is supposed to look. And since we wanted to start simple, and since the *sine qua non* of a todo item is basically just a title and a flag, here's an example of the first, launch-ready version of a valid todo item document to be stored in a user's CouchDB:

```js
{
  "_id": "todo-item:cde95c3861f9f585d5608fcd35000a7a",
  "title": "reimplement my pet project in Rust",
  "isDone": true
}
```


