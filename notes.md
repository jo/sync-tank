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


- [How can users share todo-lists? E.g. they want to build teams? Maybe todos are copied over? But how is this synced? Is it necessary to introduce a backend service?]

- [Should there be more examples now that we switch the focus to CouchDB proper?]

- What's more, if client one is online and has switched to the next version while client two is offline and still using the old version, the updated client will write documents according to the new schema. When client two comes back online it might receive documents via replication that it cannot read. End of story.
