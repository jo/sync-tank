# Distributed Migration Strategies

## Table of contents
1. Introduction
  - Who are we?
  - Focus: syncing database/app
  - What is the problem?
  - Preview

2. Concepts: Schema and Migrations
  - "Software development is change management"
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

