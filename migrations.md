# Distributed Migration Strategies

## Table of contents
1. Introduction
  - Who are we?
  - Focus: syncing database/app
  - What is the problem?
  - Preview

2. Concepts: Schema and Migrations
  - "Software development is change management"
  - distributed system?
  - schemaless? Implicit Schema
  - JSON Schema + server-side validations

3. Server-side Todo-app
  - Setup: couchdb (couch per user), website
  - Functionality: Todos with done-flag
  - Schema: title (string), status (boolean)

4. Non-breaking feature
  - New requirement: important-flag
  - Schema: important (boolean)
  - caveat: app must handle missing attributes

5. Transactional Migration
  - New requirement: enhance done to progress
  - Schema: status (string)
  - transactional (describe migration procedure)

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
  - benefits: single responsiblilty (easier to maintain and test)
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


## Introduction

Imagine you've done everything right: you've built this big, offline-first, decentralized, scalable system that supports all sorts of clients. Your agile teams are working on different parts of the software ecosystem. The database has become the glue that binds the teams together, the data structure functions as a kind of contract that all teams have agreed on. Of course, your application is live and well, customers are happy - and they demand new features.

So you have to change the data structure.

In a classical monolithic server-side architecture this is more or less a solved problem [cf...], not so for decentralized systems. Here we face two formidable challenges:

1. we cannot rely on transactions to transform data, and
2. there might be clients around that still depend on older versions of the schema.

How can we make sure different API-versions are still supported in our distributed system? How can we address the problem that there may be clients running older versions of our application after being offline for some time? How can we deal with the fact that an older application might be confronted with newer schema versions because other clients in the system have been updated already and are distributing data that adheres to these newer versions? These are among the questions we will have to answer if we want to build larger systems with enough flexibiliy to respond to change-requests.

At the time of this writing, this is still largely unexplored terrain. You will have a hard time finding articles on this topic, let alone guidelines or collections of best practices. In this article, we will make an effort to start filling this gap. In what follows, you will find a detailed exposition of our thoughts on different strategies for managing distributed data migrations. We will start with simple solutions for simple scenarios and work our way up to the complex offline-first, decentralized, multi-client, scalable systems that we are challenged to build.

Before you follow us deeper into this discussion and open your minds and hearts to what we have to say, you might want to know who we are and what we do and why you would listen to us in the first place. We are: Johannes J. Schmidt and Matthias Dumke, data architects at a company called [immmr](https://www.immmr.com/), a subsidiary of Deutsche Telekom. The company's main product is an offline capable app that brings communication to the next level as they say in the product department. It's designed to operate at a large scale in a distributed scenario. Our agile teams work on Android, iOS, desktop and web clients as well as on server side services. Our role is to ensure a seamless integration of data synchronisation between all clients, which is why we are facing on a daily basis all the above mentioned challenges of changing data structures in an offline capable multi client environment.

Moreover Johannes has a decade's worth of experience with distributed databases. He has authored and worked on several widely used tools in the Apache CouchDB ecosystem. He is the main author of the [CouchDB Best Practices](http://ehealthafrica.github.io/couchdb-best-practices/) guidelines he compiled during his work at [eHealth Africa](https://www.ehealthafrica.org/).


## Setting the stage: A toy problem

While this article is not supposed to be a tutorial, it will be instructive to have a working example at hands to illustrate some points. Our example of choice is the 'Hello World' of web applications, the todo-app. This does not look like the most daunting challenge to face from a data architecture perspective: todos can have titles and texts, maybe a creation date and a done-flag. None of this would make you sit down and write an article about different strategies for how to accomodate this information. But things get a lot more interesting once we agree to meet a few additional requirements:

1. Users should be able to edit their todos even when their internet connection is unstable or down and the application should syncronize all changes once it re-connects. In other words: the app must be offline-capable.
2. The application development should be agile, meaning that we want to support frequent changes to all aspects of the system, including the data schema.

These two requirements lead us to think about distributed migrations. Because imagine product comes up the next day with their latest idea: a todo should no longer just be open or done but should hold a number of different progress states like 'started', 'blocked', 'rejected', 'completed', etc. No problem, right? We just tweak the schema a bit, the todo state is no longer a Boolean but some form of String or Enum that can hold multiple values and we are done. - But wait, what about the todos that are already in the system? - Maybe we can update them? Or maybe the apps should be able to deal with both kinds of todos? - So we should support two schema versions now and old documents are still valid? And what if someone is using the old version of the app on their phone? The old version doesn't know anything about the new progress states. How could it possibly deal with documents of the new format? - We could force people to upgrade their apps so everybody at least agrees on the new schema. - So you want to make the entire application unusable unless someone goes to the appstore and updates to the latest version? This is not very appealing. - Do you have a better idea?

At this point we have gotten a bit ahead of ourselves and are already discussing migration strategies, albeit in a somewhat hasty and unstructured manner. But at least we understand the problem now.

This is actually a good time to take a step back and clarify some concepts that will be important throughout this whole discussion. Before we talk about schema migrations in distributed systems, let's talk about what schemas and migrations and distributed systems are in the first place.


## Basic concepts: schemas, migrations, distributed systems

If you want to store and retrieve data in an automated and efficient way, it is important to have some knowledge about the format or structure of this data. This is meta-information specifying things like data types to be used to store data. We would like to think of the **data schema** as all relevant structure-information about the different pieces of data to be stored by an application. This is a bit vague, but it get's vaguer...


outline schema:
  * a schema formalizes the structure of your data
  * Some database systems require specifying an explicit schema
  * There are schema-less databases
  * there is a way to introduce more explicit schemas (JSON schema, server side validation)
  * anyways: an application also requires data to have a particular structure
    => applications introduce an 'implicit schema'

outline migration:
  * changing the app may require change of schema
    => breaking schema-changes require a migration

outline distributed system:
  * todo example: web application with offline capable clients that store their own version of saved data
  * switch perspective: the schema as api
    => the schema introduces a dependency or contract between teams working on very different parts of the system





To get a better understanding for why this leads to problems, we propose a different way to think about the role of data-schemas. You might be used to consuming data through an API that provides access to a database. Different clients can share this API, which can also be versioned. But in our distributed scenario each client has its own database-instance. This means that in principle each client could implement its very own API to access this data. In other words: the only thing that different clients sharing the same data have in common is - the same data. To be part of the system all a client really has to do is to respect the schema. This is why we suggest it's helpful to think of the schema itself as the database API.

From this perspective, a schema-change implies an API-change for any application in our system. 

  - "Software development is change management"
