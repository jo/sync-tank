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

