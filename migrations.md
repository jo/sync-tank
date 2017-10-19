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

Imagine you've done everything right: you've built this big, offline-first, decentralized, scalable system that supports all sorts of clients. Your agile teams are working on different parts of the software ecosystem. The database has become the glue that binds the teams together, the data structure functions as a kind of contract that all teams have agreed on. Of course, your applications is live and well, customers are happy - and they demand new features.

So you have to change the data structure.

In a classical monolithic server-side architecture this is a solved problem [cf...], not so for decentralized systems. Here we face two formidable challenges:

1. we cannot rely on transactions to transform data, and
2. there might be clients around that still depend on older versions of the schema.

To get a better understanding for why this leads to problems, we propose a different way to think about the role of data-schemas. You might be used to consuming data through an API that provides access to a database. Different clients can share this API, which can also be versioned. But in our distributed scenario each client has its own database-instance. This means that in principle each client could implement its very own API to access this data. In other words: the only thing that different clients sharing the same data have in common is - the same data. To be part of the system all a client really has to do is to respect the schema. This is why we suggest it's helpful to think of the schema itself as the database API.

From this perspective, a schema-change implies an API-change for any application in our system. But how can we make sure different API-versions are still supported in our distributed system? How can we address the problem that there may be clients running older versions of our application after being offline for some time? How can we deal with the fact that an older application might be confronted with newer schema versions because other clients in the system have been updated already and are distributing data that adheres to these newer versions? These are among the questions we will have to answer if we want to build larger systems with enough flexibiliy to respond to change-requests.

At the time of this writing, this is still largely unexplored terrain. You will have a hard time finding articles on this topic, let alone guidelines or collections of best practices. In this article, we will make an effort to start filling this gap. In what follows, you will find a detailed exposition of our thoughts on different strategies for managing distributed data migrations. We will start with simple solutions for simple scenarios and work our way up to the complex offline-first, decentralized, multi-client, scalable systems that we are challenged to build.

Before you follow us deeper into this discussion and open your minds and hearts to what we have to say, you might want to know who we are and what we do and why you would listen to us in the first place. We are: @jo and @mdumke...

