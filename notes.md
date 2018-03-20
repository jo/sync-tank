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


- further points to discuss
  - we say: "the app should be **offline-capable** (what exactly this means will become clearer as we go along)" - do we actually discuss this anywhere?


- Fail fast, fail often - not mentioned so far:
  - if partially synced or ignored lots of data will be missing
  - couchdb is limited (no hooks)
  - cannot support old versions. can we switch off older versions?

- Talk about Der Semsel?
#### Der Semsel - proposal for a semantic selector in Mango

Is there a reason why the example specifies the version only with an integer after we have gone through some efforts to establish *semantic versioning* for the data schema before? Why not use the full semantic version and have `_id`-suffixes like `:v:1.2.3`? 

## relation vs relation
https://en.wikipedia.org/wiki/Relation_(database)

## materials last chapter


- lessons learned
  - make your schema explicit
  - applications should simply ignore attributes they don’t know and persist them along with the data they care about
  - whitelist the necessary documents at the access level, in particular in CouchDB views, so that new documents will not modify existing behavior


##### Versioned API

Instead of leaving old apps high and dry we may very well decide to support them at least for a while. And a common approach to implement multi-version support is through a versioned API. Imagine older clients could get older documents through the `/V1/`-API while newer clients could just talk to the `/V2/`-version. To make this possible we could still keep the CouchDB documents up to the latest version but provide server-side adapters that transform documents to the requested format.

Sounds complicated? Let's take a look at the todo app. Say someone just wrote a `todo-item-1` document to the local database of their web app. But the system has already moved on to supporting `todo-item-2` documents. Luckily, the old app uses the `/V1/`-endpoint when synchronizing `todo-item-1` documents. Behind this endpoint there is an adapter we provided that migrates the document up to the newer version and stores it. The newer clients are save! And when the old wep app wants to get the latest documents it tries to get them, of course, through the `/V1/`-endpoint. Once again, an adapter intercepts the syncing-process and migrates newer documents down to version one. The wep app is save!

As a general migration strategy, this approach looks very promising indeed. It would allow us to write adapters for each API version that could migrate documents on the fly, up and down. However, we cannot pursue this path further at this point because CouchDB does not provide any hooks that we could use to insert our adapters into the data-flow. This would have serious consequences for many aspects of the system including the replication mechanism. Since this is not an option, let's not have this discussion right now and instead focus on what is feasible.


#### Breaking changes

When we talked about schema specifications and validations, we suggested to go with semantic versioning when putting the data schema under version control. We also said that schema changes would have be *interpreted* in order to apply semantic versioning to them: What counts as a feature? And when is a change breaking? We didn't say much about this point back then because we did not have the vocabulary to discuss the topic adequately. But we do now. So let's take a look.

Put simply, a *breaking change* happens every time you modify one thing and another thing stops to work. Maybe you changed the parameters of an API and now the callers no longer know what to do with it. Or maybe you removed those unprofessional side effects from a function but another part of the program relied upon them. The situation is no different with schema changes. For instance, once you require documents to have a certain property, older version of these documents may become invalid and break newer clients.

There are several types of schema changes and we have encountered a few of them in our examples: adding new document types, adding and removing *optional* properties, and adding or removing *required* properties. In general, we can think of specifications to become *stronger* or *weaker.* Making specifications stronger means we are more demanding as to the specific format of documents, and this can happen in a number of ways: (1) by adding a new required property to a document, (2) by making existing optional properties required, and (3) by restricting the format of properties, e.g. removing available options from Enums or restricting number ranges.

* Making specifications stronger *always* introduces a breaking change.

But what about weaker specifications? *Prima facie* weakening specifications should only introduce new features ("we can now also store strings longer than 20 characters"), but this is not true for every context. If we are planning to implement a *multi-version* migration that allows us to support multiple client and schema versions in parallel we have to think *bi-directional*. It is no longer enough to make sure that newer clients can work with older documents, but also that older clients can handle newer documents. In this scenario, a weaker specification can render a new document unusable for an older app ("I cannot deal with strings longer than 20 characters").



#### The matrix

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

