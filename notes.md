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


[describe changes feed and backend services in couch-chapter]

[TBD: mention upcoming document ACLs in CouchDB and one big database as opposed to couch-per-user]


These two requirements lead us to think about distributed migrations. This may start in the product department, where product managers [obviously](https://medium.com/swlh/designers-will-design-developers-will-develop-and-why-you-must-stop-them-399255275593) want to evolve the app into a full-fledged project management suite. Product decides that a todo should no longer just be open or done but should hold a number of different progress states like 'started', 'blocked', 'rejected', 'completed', etc. No problem, right? We just tweak the schema a bit, the todo state is no longer a Boolean but some form of String or Enum that can hold multiple values and we are done. - But wait, what about the todos that are already in the system? - Maybe we can update them? Or maybe the apps should be able to deal with both kinds of todos? - So we should support two schema versions now and old documents are still valid? And what if someone is using the old version of the app on their phone? The old version doesn't know anything about the new progress states. How could it possibly deal with documents of the new format? - We could force people to upgrade their apps so everybody at least agrees on the new schema. - So you want to make the entire application unusable unless someone goes to the appstore and updates to the latest version? This is not very appealing. - Do you have a better idea?
In fact, we do. Stay tuned.


At this point we have gotten a bit ahead of ourselves and are already discussing migration strategies, albeit in a somewhat hasty and unstructured manner. So let's roll up our sleeves and start building out this app, step by step.

#### grammar questions
- This would for instance blah... vs This would, for instance, blah...
- todo item vs todo-item
- todo app vs todo-app
- data schema vs dataschema
- client-side vs client side
- multi-version support vs multi version-support vs multi-version support vs multi version support
- user database vs user-database
- of course, blah vs of course blah
- However blah... vs However, blah...
- `id`-attribute vs `id` attribute
- set up vs setup
- Be careful, though, not to... vs Be careful though, not to...



- Fail fast, fail often - not mentioned so far:
  - if partially synced or ignored lots of data will be missing
  - couchdb is limited (no hooks)
  - cannot support old versions. can we switch off older versions?

- Talk about Der Semsel?
#### Der Semsel - proposal for a semantic selector in Mango

Is there a reason why the example specifies the version only with an integer after we have gone through some efforts to establish *semantic versioning* for the data schema before? Why not use the full semantic version and have `_id`-suffixes like `:v:1.2.3`? 




## materials last chapter


