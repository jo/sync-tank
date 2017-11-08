### concepts
- implicit schemas / structure through dependence, the app defines the schema
  -> refer to JSON-schema as a way to make the schema explicit
- migration: transformation of data from one structure into another


### problem exposition
- easy migration: 1 application, 1 database
  -> transaction with rollback
- context:
  * more than 1 db
  * distributed dbs (no control over updates)
  * distributed apps (no control over updates)
  * no transactions

## Todo
- link to best-practices post
- Don't make this a tutorial


## Validation
validation of multi schemas

## Schema
## v1
{
  _id: "todo-item:12312342132",
  text: "Clean the dishes",
  isDone: false
}

## v2
{
  _id: "todo-item:12312342132",
  text: "Clean the dishes",
  status: "progress"
}

## v3
{
  _id: "todo-item:12312342132:status",
  status: "progress"
}


- [do we keep the version in the doc-type?]

- [How can users share todo-lists? E.g. they want to build teams? Maybe todos are copied over? But how is this synced? Is it necessary to introduce a backend service?]

- [Should there be more examples now that we switch the focus to CouchDB proper?]

- live migration:
  * caveat? Document conflicts when new app is writing new documents?
  * does reading old versions and writing new ones not lead to weird datastates (like not updated old docs that will have to be ignored for all time)

- maybe move the accounting discussion into its own aside because it may be lengthy and relevant across migration strategies

- The database has become the glue that binds the teams together, the data structure functions as a kind of contract that all teams have agreed on. 

- Sometimes offline-capability is not just nice to have, but a hard requirement -> mention immmr again

- move Couch-introduction from todo-app chapter to couch-chaper
