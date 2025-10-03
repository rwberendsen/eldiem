# Declarative schema management

From the engineering community we have a great push towards everything as code:
infra, networking, configuration, and so on. However, in the data community it
is still common to have manual schema management, even manual data writes,
manual data loads, and so on. There seems to be a large culture gap between the
communities.

One thing that an analytics team has to deal with is long term storage of
historical data. While data models, business processes, service offerings, and
the world keeps changing, analytics teams may have to answer questions years
after the fact, for example about financial performance indicators. 

In engineering, there is a strong “hands-off production” culture, but, in
databases, and in particular in analytics, we see a strong wish coming from
people to stay in control, to have admin rights in production, and manually
correct data, re-load data, delete data, and so on. But, are the same security
considerations for the engineering culture not valid for data as well? The same
auditors that demand such a strong separation between non-production and
production environments in IT, would they not demand as well that in analytics
data administrators should not manually perform write operations in production?

This brings me to what production means. When is software reaching a production
ready state? In my opinion, this is only when enough thought has been given to
how future changes are going to affect customers (internal or external) who are
already depending on what is currently in production. Wild and backwards
incompatible schema changes are a sign software is not reaching production
stability yet. Analytics teams are internal consumers of data produced by
business processes and transactional apllications. In real life, even in
production crazy schema changes, data corrections, and data migrations happen
on the engineering side, and analytics teams have to collect all this data and
present it back to the business as if it all made sense.

Where in the analytics domain administrators are able to manually catch
odd-balls, make schema changes manually in production, taking care and doing
hard thinking on how not to destroy already loaded data, they can adjust to
almost any situation; only, when data teams get larger, it will be harder to
oversee why everything is the way it is, who changed what, when, for what
reason.

What if we demanded in analytics that no manual writes and / or schema changes
could be performed? That everything had to be done in code? There are broadly
two approaches to any “whatever as code” methodology: imperative and
declarative. 

In data engineering, as in engineering, there is a lot of experience with the
imperative methodology, with tools like, e.g., FlyWayDB. On the analytics side,
where Snowflake is one of the big players, we have a tool like schemachange, a
schema migration tool developed in Snowflake Labs. A pro of imperative is that
it is very easy to implement, you can code everything that you could run
manually. A con of imperative is that it reads very much like an audit log. You
cannot easily deduce the current state of your configuration from it. And then,
if you migrate to a new schema migration tool? What do you do, rewrite the
whole history in this other schema migration tool?  Moreover, in analytics,
Snowflake and Databricks, large players, are much more than database engines
alone.  They are complete and feature rich platforms for different kinds of
data teams, e.g., data scientists, analytics engineers, ETL engineers,
datawarehouse engineers, and so on. Their platform keeps evolving and changing
fast, to keep pace with every more and varied user demands, features, and to
keep pace with security requirements.  With such platforms, you cannot possibly
expect to run a script that is a few years old now, on a brand new Snowflake or
Databricks environment. Thus, one of the motivations for infra as code
(reproducibility) gets rendered meaningless.

With the declarative methodology, there is a ton of experience in the
engineering community, on top of cloud platforms for example, with tools like
AWS CloudFormation, Terraform, OpenTofu; or on K8s, with K8s manifest files,
Helm chars, Ansible playbooks and roles, (some people call Ansible imperative,
but, I would say it is an attempt at a declarative technology where the author
of a plugin needs to ensure idempotency his or herself). A benefit of
declarative approaches is that at any moment in time, your declarative and
version controlled configuration is expected to reflect the actual
configuration. A downside of it is that the engines that are responsible for
making this happen are hard to implement.  Typically, these are not in-house
software engineering products. AWS CloudFormation is a service developed within
AWS. Terraform is a large open source product (well, OpenTofu is), with
providers that are themselves large engineering efforts. These engines need to
keep evolving to stay up to date with the platform they are managing resources
on. There are always cases where emergency fixes and manual changes need to be
done in production, and then that results in drift between your declaration and
the reality, and engines may or may not be capable of dealing with and
repairing that drift. Also there are always bootstrap actions that need to be
done outside of the declarative engines, and engines need to be able to deal
with that as well.

Do we have declarative approaches when it comes to managing analytics data
stores, data warehouses, and data marts, data lakes? When it comes to the
schema of the data stored there, the logical data models? In engineering we do
have schema registries, schema repositories, and these technologies are used 
in event streaming architectures mostly. Event warehousing is also a practice
in analytics. But with long term data storage the danger is always the
state: the data that is already in the systems, in the tables, in the views. 
Any declarative engine could never be allowed to corrupt any of that data. Also,
while transactional applications have to respond correctly to current data points,
analytics logic has to correctly handle data points generated over a span of
many years, in many cases. Iceberg tables, Hudi, Delta tables, the analytics space
has also seen major innovation around managing schema evolution over time. 
Is there still room for improvements? For a way of metadata management of logical
data models that is not tied to any particular technology?

To explore possible approaches here, we need a few definitions. Even if we
would not create anything new here, these concepts will help us discuss current
innovations in the analytics space, and categorise them. Typically, *backward
compatibility* is defined in terms of changes to a schema used by an
application to parse incoming data: it is backward compatible if an application
using the new schema would still accept old data as valid. When we talk about
analytics, we can use the definition that downstream queries, when adapted to
use the new schema, would accept old data as valid, then the schema change is
backward compatible. *Forward compatibility* can be defined as when an
application is using the old schema, it would accept as valid data that was
produced using the new schema. In analytics, this would mean that downstream
queries would remain correct without changing them, even after the schema
change.

## Schema of data vs schema of data store

Can an analytics application that is aware of the schema of a logical data
model manage a data storage layer and its schema automatically, safely? To
proceed, we need to make a very important distinction here: *the schema of the
data itself, and the schema of the data store*. The data store is simply a
container, and we are interested in managing its schema declaratively, using
logical data models of the data that we are writing into it. For an approach
like this to work and enable correct, schema-aware analytics, every data point
written should be tagged with its schema version. Ideally, this is generated by
the application that produced the data. In the real, gory data landscape, data
teams may have to generate and manage these schema changes in many cases. 

Then, if a declarative engine is used, as long as it is possible to alter the
storage schema in such a way that it is backward compatible with all schemas of
data points that were written in it, the schema can be altered without
intervention. So both backward- and forward compatible schema changes would be
acceptable. Even schema changes that are neither backward- or forward
compatible can be accomodated in this way, e.g., adding a required field. But
changing the type of a VARCHAR field to an INTEGER (or the other way around) is
something that would be *destructive*: *there is no possible schema that can be
backward compatible with these two schemas*. So it seems to be a reasonable
policy for an enterprise wide way of working regarding logical data models that
*if a destructive change is made, then a new name for the logical data model
must be used.* Then, applications everywhere will be enabled to create fresh
storage layers for data written with the schema of the new logical data model.

To see why we need this distinction between the storage layer its schema and
the schema of the data, consider what happens when we add a non-required
column:  yes, old rows are still valid in the new schema. But there can be a
difference in how to interpret an old row that was written when this column did
not yet exist; and a new row that was written after the column existed but that
omitted supplying a value for the column. In some cases, correct business logic
may need to incorporate and handle differently data that was produced with
different schema versions. This is perhaps the main complicating factor for
analytics, compared to engineering. Transactional applications typically only
have to handle correctly current and recent schemas. In real data life, this is
when we see timestamps in analytics query, where data written before and after
a certain time is handled differently. Of course, this is terribly hairy, it
would be a major improvement if schema versions could be used instead of
timestamps. So, we need to know the schema that the producer of the data row
was using when it produced the data, ideally. And this schema definition needs
to be available years later, say, in a schema repository. That would be a first
necessary step for still being able to correctly interpret a data point years
after the fact.  Unfortunately, this simple requirement is already like a holy
grail in the data world. 

## Schema of events vs schema of state

Another fundamental difference in engineering, which plays a crucial role in
event driven architectures, is the the schema of events, versus the schema of
state. As a very simple example, an event might be a transaction where an amount
of money, say 100 euros, is transferred from bank account A to bank account B.
If the transaction is accepted, an event is created, and it may log this exact
information. However, how to decide whether or not to accept the transaction? For
this, an application needs state. For example, the current balance of account A,
and the minimum balance allowed at the moment. The value of the information in
the event is therefore very closely intertwined with having available the exact
state at the time the event was accepted (committed). In an ideal world, having
an append-only and error-free log of committed events would allow downstream
teams to reconstruct exactly the state at each moment in history, without error.
In practice, however, things can get quite complicated. Bring in distributed
systems, and eventual consistency, and you get the idea. 

In many enterprises, event driven architectures may not yet dominate the game.
And in many analytics teams, primarily, state is what is being transferred. In
the bank account example, we'd get the balance of accounts. Perhaps we'd only
get balances of accounts that had been modified since we last pulled data. We
are getting data from sources that update their rows when events happen. And,
downstream, we may end up updating rows in tables as well.

How does this work, then, if between the time a row was created, and between
the time a row is updated, the schema of incoming data has changed? What schema
should we say a row like this was produced with?

In cases like this, many times incoming complete rows, complete with state
information, are often referred to as events. Or maybe events carrying state
information. So, an event of a transaction may include information about the
state of both accounts at the time of the event, and any applicable limits at
the time, so that all information used when the event was produced is carried
over in the event data. Or a record comes in from the account table, and we get
only changed records, and for these records we get the balance before and also
after the update, with some info on the last transaction. And in a staging area
of our analytics domain, we can in fact store such incoming records in an
append-only fashion, allowing us to replay these updates after the fact at
will, if we messed up something in downstream tables where we are updating
state.

And what about these downstream tables? The schema in there really is owned by
data teams responsible for transformations, computation of KPIs, and analytics
business logic. If upstream staging tables have seen forward compatible changes
such as previously allowed values that are no longer allowed, or, numerical
measures that have a lower precision, these teams have to make hard choices
what to do with historical data. For example, truncate historical numerical
values? Round them? Or present different reports for time periods where the
historical schema was used? Arguably, when rows are updated in such tables, the
schema of the data last used to perform this update carries most weight.
Still, it may have value to actually maintain a list of schema versions used by
each update. Note the the YAML format we propose in this document does not
prescribe such behaviour or provide any implementation help for it. We merely
sketch how managing machine- and human readable logical data model schemas
centrally could help developers all over the enterprise to build more robust,
resilient, and correct pipelines and applications.  Such stateful tables, too,
may have columns that are no longer in use. So, still, the distinction between
the table of the data store and the data itself has value. 

When it comes time to expose data to downstream consumers such as data kubes,
data virtualisation platforms, or, more commonly, data visualisation tools, 
there is still value in calling out a current schema for that interface; which
may differ from the schema of the physical table(s) holding the data.

## Logical vs physical data model catalogs

The distinction between the logical data model schema version that the data was
produced under and the physical schema of the storage layer is also important
for other reasons. For example, we may expect a STRING(20) based on a logical
data model, but use a STRING (of unlimited length) in the physical schema of
our analytics data store, because in our physical storage layer, there is no
benefit to capping the length of STRINGS, and in case the producer would
increase the length of the field, its simpler if we don’t have to do an ALTER
TABLE. Even if we did cap the length in our physical data model, if the
producer would shorten the length of the field (forward compatible schema
change), we would not, because it would destroy data written under the old
schema). So, the schema of the physical storage actually does not tell us
anything anymore about the schema of the data contained in it. If we drop the
information of the schema of the logical data model, and we do not cap the
length of the STRING field at all, then any consumer downstream has to have the
ability to store strings of arbitrary length; or, come up themselves with an
accurate cut-off point, reinventing the same information. It goes still deeper
than this. What does a STRING length of 20 mean? In some databases, it may mean
bytes. And for unicode characters, one character may occupy four bytes,
depending on the encoding. Okay, so perhaps we should aim for a logical
interpretation in our logical data models here, and say that 20 corresponds to
20 characters, perhaps decoded Unicode code points. While a physical data model
is free to deviate from that, based on the specifics of the database engine
used.

In practice, in many cases, the distinction between a physical data model
and a logical data model as found in catalogs is not used. Instead, catalogs
are tightly connected to database engines. For a typical database engine, the
DDL statements themselves constitute the catalog. What you find in them is
simply the current schema of physical storage tables. Many data pipelines
already alter these schema’s on the fly in response to changing source tables.
These data pipelines directly map physical types from the source database
engine to physical types in the destination storage layer. Analytics queries
using the destination tables often have no way of knowing that old data was
written prior to the existence of some of the columns. The situation is only
somewhat workable because SQL is a very stable language, and the ANSI SQL
standard goes a long way, database engines have variations of types, but
translating between these type systems is fairly straightforward 90% of the
time. The other 10% of the time it hurts, when encoding issues pop up, or
different handling of NULL values. In some catalogs, tables whose schemas are
not altered on the fly are called “governed tables”. Here, the idea is that
administrators come in and alter the schema manually, e.g., by issuing ALTER
TABLE statements, or, by adding field in a catalog UI. After that, pipelines
that failed before are expected to resume correctly. The end result is the
same: the catalog ends up holding the current version of the schema of the
physical storage layer. Finally, a product like Collibra has metadata
collection engines that can connect to a wide variety of such database
catalogs, and it will sync these catalogs with their own catalog (requiring
also a single type system able to represent a wide variety of type systems). In
some cases, these engines generate conflicts, for example when in one schema
change a field is removed, and another field with the same type is added,
Collibra would request a human administrator to resolve a conflict: was a field
renamed, or was a column added and the other one removed? Again, the end result
is that we have at all times a catalog of current (and current-only) physical
storage layer schemas.

## Imagining real life data pipelines using schema repositories

Many data pipelines, operating in the dark, simply harvest raw data from a data
layer deep down in some vendor its product, almost illegally connecting
straight to some database tables and just dumping their contents in some
staging area. When the vendor changes their product, their database design may
change, and there would be no schema registry in sight. In such cases, it may
make sense for data teams to use a “consuming data model”, i.e., they would
themselves clearly define the logical schema of the data as they would expect
it to exist in the source, and version that themselves. A *tolerant reader*
would then accept it if in addition to the expected data other data fields
emerge in the source; a *strict reader* may refuse to load data because who
knows after such schema changes in the source the old data points take on a
different meaning? But really, it would be hard to imagine a correct strict
reader that would detect all such cases. 

Even if we have a schema registry, and for awkward data sources (the majority
in any given enterprise setting: CSV files, JDBC connections to various
databases, SFTP file transfers) data teams would manage schemas declaratively
themselves, managing schema’s declaratively over a span of years may prove
elusive. First, what language should be used to describe schema’s? This
language should be highly stable. It should accurately describe schemas that
may exist in a variety of platforms, databases, file formats, e.g., PostgresDB
databases, Parquet files, ORC files, Thrift, Protocol, JSON, CSV , and unknown
future technologies. 

Second, what technology would we need to store the logical data models. To not
pin anyone down on anything (no more vendor lock-in, please), it seems that we
should go for a plain text file format. This file format should be extremely
stable if it needs to be used for years. It should be both machine and human
readable. It should be easy to version control it. It should support comments.
YAML comes to mind, despite its imperfections; but YAML would just be the
format for the language; the language itself would have to be designed, a
grammar for it.

So, we would need a language independent of any physical storage format, and
with an ecosystem of software that could be aware of this schema language, and
translate this to appropriate physical data models for the technology they work
with. For example, read in a schema from our registry, and translate that to a
DDL for PostgresDB, and, the other way around as well. 

And logic and policies that can work with this new schema language, and compare
schema versions, and decide if a proposed change would be backwards compatible
or not. And we would need lineage between schema versions. The schema language
would have to be rich enough to allow engines for a vast array of platforms to
manage physical schemas in a declarative fashion based on logical schema
evolutions, in such a way that performant schema’s are used in the platform of
choice.

Surely a daunting task. Json-schema comes closest of any technology I have
seen. XSD perhaps even closer, but, it seems unlikely folks would want to go
back to that. Do we need something new? YAML based?

## A sketch of a possible schema language

For analytics, perhaps we couls use something new? Something based on the
concepts of collections and constraints.

- Scalar types: 
  - String
    - With max length, optionally
    - With allowed values, optionally
  - Numeric
    - With precision and scale
  - Float
    - With tolerance, a float value with the maximal difference for two floats to still be considered equal
  - Boolean
  - Dates?
- Struct type, having named fields of any type
  - With required constraint on each field
  - Embedded structs, inspired by the Go programming language, to model composition
  - A flag on how to treat unexpected fields: FAIL, IGNORE, STORE, more on this below
- OneOfStruct type, to model inheritance
  - We require provably non-overlapping constraints on a set of fields that is present in all alternatives
- Collection of scalar types
  - With uniqueness constraint (making the collection a SET (of scalars))
  - Note that even floats can be compared if we have defined tolerance)
  - With ordered constraint (meaning elements should not be reordered; their order is significant). This would model an ARRAY or LIST type.
- Collection of struct types
  - With constraints on collection: such as combination of struct fields that yield a unique row. Note that in this way we can model TABLEs, but, also MAPs. The combination of struct fields that make a row unique would represent the key of the map; the remainder of struct fields would represent the value. If there are no remaining fields, we have modeled a SET (of structs).
  - With ordered constraint (meaning elements should not be reordered, their order is significant).
  - With constraints on relations with other collections, such as relationship between fields in this struct to fields in other structs. This can be used to model things like foreign key constraints.

The flag indicating how unexepted fields in a struct are to be treated is related to the concepts of a tolerant reader which
does not fail if unexpected fields are present and a strict reader that would fail in this case. Using the flag the publisher
of the LDM can specify on the level of individual (nested) structs how readers are requested to behave:

- FAIL: The publisher promises no unexpected fields will be there, if there are, readers should fail.
- IGNORE: The publisher promises that while there may be unexepcted, additional fields, they should not be of interest to any downstream team, they carry
no meaning related to any actual business process, and they can safely be ignored.  
- STORE: The publisher requests readers to store unexpected fields, since these fields do carry information that may be needed downstream, but the schema
cannot be known in advance, or it is impractical to specify it in advance. In this case, the publisher can never add new fields or rename existing fields, 
because their names and / or types might conflict with fields that readers have already stored.

With just these concepts, I believe we could express a lot, from the nested
tables we can have in BigQuery, to documents in a document database, from JSON
objects to Parquet files, and so on. It is possible that a separate Map type
would still be useful, to lift the distinction from using a map or a table to
the logical data model level. If not done, then engines would have to be able
to recognize on the fly data that is in a physical map in some storage layer
should be related to a collection of structs with a unique key constraint; and
the other way around when presented with a collection of structs in a logical
data model, they have to decide to use a table or a map in the physical storage
layer they happen to manage.

## A sketch of a possible format for the schema language

While we suggested YAML several times in this document, YAML, too, has its pros
and cons.  It is a rather rich language, that actually was intended to contain
data, much like JSON, which is simpler, and is considered a subset of YAML. Our
business is mainly specifying data types. We may go for something more
specialised. SQL itself comes to mind, the golden standard for data querying
and processing in analytics: SQL style DDL statements. Or any statically
compiled programming language, too, has syntax for specifying data types. 
A recent and popular language developed by some of the smartest brains in computer
science is Go. We can take inspiration from its syntax to define our schema
language syntax.  Rust is another recent language, low-level, trusted to be used
in the Linux kernel code base, and open source. Most likely, we will use Rust as a first
choice to write a parser for our schema language. Still, we are free to use
any style for our own schema language. Say we take inspiration from Go, then our format
could like something like:

```
namespace: com.example.corp.eldiem/line_of_business/v1
version: 1
compatibility_mode: backward

type contract struct{
  id string !nil & maxLength = 24
  customer_id string !nil & maxLength = 24
  start_date date !nil
  end_date 
} end_date = nil | end_date > start_date

type customer struct{
  id string (
      !nil
    & maxLength = 24
  )
  customer_email string !nil & maxLength = 320
}

contracts [contract] uniqueKey([id]) & foreignKey([(customer_id, customers.id)])

customers [customer] uniqueKey([id])
```

In this small sketch there are already quite a few things to digest. First,
we start with a namespace. It follows the format common in namespaces of classes
in Java, with reversed domain names. Then follows a slash, which you can further
use to group names and types together. Later we will see that also names can have
path elements in them, which you can use as a further hint for how teams may group
names together, in case you have large logical data models. Note also that we
use a v1 in the namespace here. It is just a naming convention we would recommend,
but not required or enforced in any way, we'll get to the reason for this convention
in a few paragraphs.

A namespace corresponds to a single logical data model (LDM). It is the unit
that you version. The first version is always 1. Whenever you evolve an LDM,
you will bump this version number by 1.

The compatibility mode specifies for this LDM how it may be evolved. This cannot
be changed during the lifetime of the LDM. That means that the compatibility will
always be transitive over multiple versions. Note that in no event a destructive
schema evolution will be supported: this will require using a new namespace.
And this is why it is useful to end your namespaces with an explicit version as 
we did above. If you want to make a breaking change, but you still want to use
a similar name, and logically replace your earlier `line_of_business` namespace, you can simply
create an new namespace: 

```
namespace: com.example.corp.eldiem/line_of_business/v2
```

After the header, we get types and names. If its a type, it follows this grammar:

```
'type' type_name type_def
```

If it's a name, it follow this grammar:

```
name ( type_name | type_def )
```

With
```
type_def: 
    'string' [string_constraints]
  | 'numeric' [numeric_constraints]
  | 'bool' [bool_constraints]
  | 'date' [date_constraints]
  | struct_type_def [struct_type_constraints]
  | one_of_struct_type_def
  | list_of_scalars_type_def 
  | list_of_structs_type_def [list_of_structs_constraints]
  | list_of_list_type_def [list_of_list_constraints]
  | map_type_def [map_constraints]
```

Now in the above grammar snippets, we again find a lot to unpack. Note how
different types allow different types of constraints. We briefly touch upon
some aspects.

`list_of_scalars_type_def` omits optional constraints because it will be
further expanded to different types of scalars in order to allow appropriate
constraint expressions depending on the scalar type that is used.

`one_of_struct_type_def` would have constraints embedded too, 
as we can most easily show with an example first:

```
type order struct {
  order_id string !nil
  offer_id string !nil
}

type event struct {
  event_id string !nil
  event_type string allowed(["OrderPlaced", "ContractSigned"])
  payload oneOf [
    orderPlacedEvent event_type = "OrderPlaced",
    contractSignedEvent event_type = "ContractSigned"
  ] 
}
```

Here we see that the constraints on the list of alternatives are
indeed mutually exclusive, disjoint. By making this a requirement
we can ensure that a validation process that is using this LDM to validate
an instance will be able to determine the intended schema of the instance
that it is validating. Is there a good use case for JSON schema
it's anyOf and allOf features? If we come across one, we can expand our
format.

Now, what happens when we want to evolve a model? In analytics, tables can
easily have hundreds of columns. We don't want to rewrite our entire LDM
if all we need to do is change the size of a column. For that reason, we need
our schema language to support manipulations. First, let's see an example.

```
namespace: com.example.corp.eldiem/line_of_business/v1
version: 2

create {
}

update {
  type customer.id string !nil & maxLength = 48 // backward compatible change
}

delete {
}
```

Finally, we can shadow another LDM:

```
namespace: com.example.corp.eldiem/dwh/staging/line_of_business/v1
version: 1
shadows: com.example.corp.eldiem/line_of_business/v1
sourceVersion: 1

create {
  type customer.valid_from date !nil
  type customer.valid_to date 
  type customer.load_date datetime !nil

  // same for contract type
  // ...
}

update {
  typeConstraint customer valid_to = nil | valid_from < valid_to
  nameConstraint customers uniqueKey([id, valid_from])

  // similar for contract type and contracts name
  // ...
}
```
Shadows inherit compatibility mode from their source LDMs.

Note how we use keywords like `type`, `typeConstraint` and `nameConstraint` to
denote what we are changing exactly. Also note how we are using dot notation
to traverse from the a struct type down to its field. In fact, this is an example
of a  mini query language that can be used to identify even nested fields.
We did see an earlier example that in our usage of a `foreignKey` constraint, 
where we referenced the name `customers.id` to denote the `id` field of the struct
`customer`. The full syntax for that reference would contain a splat expression:
`customers.[*].id` meaning that we denote the id field of all the struct records
in the customers list (table). When using the dot notation on a list of structs,
the splat expression is the default.

What happens if the source LDM issues a new version? Downstream, we publish
a corresponding new version:

```
namespace: com.example.corp.eldiem/dwh/staging/line_of_business/v1
version: 2
sourceVersion: 2
```

This could mean a conflict, if the source modified a constraint that we
modified also, for example: in such cases, implementations should not accept a
file like this without modifications. In this case, we need to spell out
create, update, and delete manipulations with regard to our own version 1.  But
if there are no conflicts, the same create, update and delete actions that were
done in the source LDM are implied here, too.

And if we want to change our downstream model? Then we only bump our own version:

```
namespace: com.example.corp.eldiem/dwh/staging/line_of_business/v1
version: 3
sourceVersion: 2

// ...
```

We specify the changes to our LDM with respect to version 2.


## Schema repositories

To manage ancestry of logical data models, we could propose two mechanisms.
First, LDMs have a globally unique name (enterprise wide). This can be
accomplished also by grouping them in namespaces corresponding, e.g., to
domains, or lines of business, such that LDMs can be unique within their
namespace only. The first version is simply version 1. Everytime a change is
made, this version number is incremented. In this way we know that 2 will be a
child of 1, 3 will be a child of 2, and so on.

The repository should at all times contain the past as well as the current
situation. Therefore, it should be *append only*. Rather than requiring full
duplication of an LDM (which can have many tables inside it) for every little
change, the spec will offer a way to declare changes (making it look like an
imperative approach; but in reality still declarative). For example:

- Adding a field to a struct type
- Altering a constraint, e.g., making a non required field required
- Removing a field
- Renaming a field

The last one is interesting, and a clear example of a situation in which
downstream analytics queries need to be aware of schema changes.

Second, we offer a complementary mechanism. Here, we allow explicit declaration
of dependence on a logical data model with a different name. This way, when an
engine is copying over some data and simply adding some metadata fields to each
table, it can declare its own logical data model as depending on the logical
data model of the data in the source, and declare some changes containing the
fields it is adding. If the source logical data model its version is bumped,
the engine can bump its own version. This new version now has two parents: the
explicit dependency on the new version of the upstream logical data model (with
the change being adding the engine its own metadata columns), and the the
dependency on the previous version of the engine its own logical data model
(with the change being the same change that was done in the upstream logical
data model.).

In this way, the physical storage layer for the repository can be as simple as
a collection of files, i.e., a hard drive is all it takes to store it. 

Still, there are hard questions on who or what is allowed to make changes here.
If we allow engines to fully automatically publish new logical data models,
then first of all we need an API for these processes, and second we need ways
to prevent conflicts and inconsistent data. Another option could be to use
GitOps ideas and require merge request reviews for every change; blocking
pipelines potentially, and risking data loss. 

The latter approsch would be the primarily
envisioned way of working in the context of which the ideas presented here matured.
A central (collection of) schema repository / repositories to govern integrations
between APIs offered and managed by different teams; whether those APIs would
be REST APIs or simply database views exposed over a JDBC connection; or file
transfers for that matter. Where the idea would be that teams that expose the
API / database objects / files are responsible for publishing the logical data
model of the data they offer. Such a logical data model could then be used to
attach metadata to relating to matters such as data classification, privacy,
data ownership, production vs non-production status, and so on; it can be a 
component in a wider governance framework, and it can be a component in a
data-mesh way of working, for example. In such a context, we can imagine it
as part of a standard process that if a data producer would want to do a schema
change, they would first publish the new version of their logical data model
centrally, giving consuming teams time to, for example, already prepare their
underlying data stores (automatically), to be able to hold data of both the old
and the new schema. After this, the consuming teams would still be writing data
under the old schema in the altered data stores, until upstream teams actually begin
publishing data with the new schema and consumers actually start consuming these
APIs. This would allow for performing backward compatible schema changes without
breaking pipelines. 

## Related work

As noted, projects like Iceberg, Hudi, and Delta Lake represent vast innovation
efforts in the analytics space. How can we relate them to the definitions and
ideas expressed above?

Note that our proposal here of managing logical data models in a LDM language
expressed in YAML is a bit further away from real data processing engines and
storage layers than these technologies. We propose that engines or rather even
applications built on top of compute platforms could be aware of logical data
models, and use that metadata to inform how to manage physical tables. Iceberg,
Delta tables, and Hudi are far more and deeper integrated with actual compute
platforms and storage layers themselves. An application would use Iceberg or
Delta tables or Hudi tables, and use APIs exposed by these respective technologies to
manipulate them. In contrast, applications that know how to consume YAML files
with logical data models may still use Iceberg, Delta tables, or Hudi tables,
or PostgresDB tables, or MSSQL Server tables, or CSV files, or JSON files. And
the application would be responsible for using the respective APIs still to 
actually manipulate schemas of physical objects. The question is more: given the
advanced capabilities of schema evolution of modern analytics compute frameworks
such as Iceberg, Hudi, and Delta tables, is there still a need for overarching
schema managment of logical data models?

Also note that Iceberg, Hudi, and Delta do not actually offer declarative
schema management. What they do offer is, at the cause of some operational risk,
responsive, on-the-fly adaptation of existing schemas, in reaction to
schema changes in upstream sources. In a way, this approach contrasts with the
desire behind managing resources in a declarative way; the latter approach stems
from a desire to oversee and govern and control the state of resources with
confidence; whereas the former approach emphasizes a continuous flow of data
and little human intervention, greater flexibility, perhaps at the cost of
some control and governance.

Closer to the current proposal would be work on data contracts, and work done
at startups like Gable. 

### Iceberg

Iceberg tables come with advanced concepts and with deep integrations with a
wide variety of compute engines. Their scope far exceeds managing schemas of
tables. Indeed, the aims of Iceberg tables include bringing ACID properties to
table updates when tables are stored as files (e.g., Parquet or ORC) on
distributed file storage. ACID properties are limited to individual tables, however,
unlike transactions in more traditional RDBMSes like PostgresDB, where serializable
isolation between transactions can span many statements manipulating many tables. 

Iceberg is well suited for compute platforms that separate storage from
compute, such as Spark, but also Snowflake. The Iceberg spec details a very
rich metadata format for rows, partitions, tables, snapshots, and so on.
Compute engines like Spark have developed Iceberg compatible libraries that use
Iceberg metadata for essential activities such as creating query plans. Also,
they can use Iceberg catalogs to manage metadata of tables.

Iceberg comes with a rich set of data types, and normative mappings between
that type system and type systems used in Parquet, ORC, JSON, etcetera.

Iceberg tables support evolution, and the exact operations supported are
carefully defined such that existing data can never be destroyed. In fact, only
backward compatible schema changes are allowed. The reason is that the schema
that is being evolved is indeed the schema of the table holding the data. It is
the schema of the data store. By using the rich metadata stored even for each
row, such as the commit snapshot id that last updated a row, we may be able
to find the schema id and the schema definition of the table at the time that
particular row was written. But Iceberg does not address managing the schema
of the data that was written. So its scope is far wider than our proposal here
on the one hand, on the other hand, it does not cover our proposal completely.
Also, our proposal seems more technology agnostic, given the deep integrations
between Iceberg and execution engines. This is not a good or a bad thing either
way. These deep integrations are also a sign of the huge success of the Iceberg
format.

In our proposal above, we also allow forward compatible schema changes. However,
this is because we are primarily interested in the schema of the data that is
being inserted in a table; not in the schema of the containing table itself. 
Of course, also in our proposal, physical table schemas can only be updated in
a non-destructive manner. One small difference between Iceberg and our proposal
here is that we would not allow re-using a column name after it has been deleted;
most definitely not when that new column has a different type. In Iceberg this
is possible because internally, columns are identified by globally unique identifiers.
But, consumers use column names when they query tables. If they refer to a re-used
name, I can only imagine that it would be hard to retrieve historical data for
previously existing columns that carried the same name. If in our proposal, a schema
registry for a logical data model would be configured in such a way that bringing
back to life a deleted column is allowed, then, a query using it would also bring
back historical data for that column, from previous life cycles. This would be
possible because the data would have the same type (possiblly demoted or promoted,
but still).

### Delta tables

Delta tables also come with ACID properties for concurrent read and write operations
on individual tables. A transation log keeps track of updates to a table, and rich
metadata is provided out of the box, much like it is the case with Iceberg tables.

Schema evolution of delta tables is perhaps a bit more tradtional, when data used
to perform inserts or upserts has a new schema, Spark exposes APIs to change the
schema of the update table on the fly, supporting limited options like adding new
columns or perhaps promoting the type of existing columns to allow a broader set
of values. Deleting columns, or reordering them would require issuing ALTER TABLE
statements.

### Data contracts

Data contracts are an emerging topic to handle change management in transactional
applications with regard to the impact of such changes in downstream data products.
Sanderson et al have done considerable work on describing ways of working with data
contracts, and also go into some detail on how an implementation of them might look.
One component of a data contract would be a schema speficication, for which they
recommend JSON schema. Work by Andrew Jones considers also Avro, and Protobuf, as 
well as JSON schema. Indeed, this document is most closely related to work on data
contracts, and much of the discussion, also in relation to technologies such as
Iceberg and Delta Lake would apply to any approach one would take when working with
data contracts.

## Discussion

So what we propose here is a data format that is human and machine readable for
capturing logical data models and schemas. This would organizations to start being
more explict about data exchanged between teams. The easiest way of using it
would be to have a human-in-the-loop when altering an LDM, and a simple git
repository with a peer review process may be enough; but within data management
frameworks other uses could be employed. The spec would also contain rules for
evaluating exactly what would constitute allowed schema changes depending on
if backward- and / or forward-compatibility was demanded for evolving an LDM.
Probably implementations of these rules would also be developed, for a variety
of programming languages; organisations would run these themselves, for example,
they could be configured to run in response to creations of merge requests.

Applications in the organization would then be able to use the LDM catalog to
manage data pipelines and data stores. In the case where advanced platforms
like Iceberg or Hudi or Delta Lake or Snowflake are used, which have their own
catalogs, and which already allow semi-automatic schema changes, then still we
maintain that an Icebberg catalog, for example, is a catalog of current schemas
for physical data stores, despite the fact that Iceberg has its own set of data
types, which map to data types of underlying storage formats, such as Parquet.
We propose yet another layer of abstraction on top, and yet another type system
that would facilitate logic to reason about what consitute back- and / or
forward-compatible schema changes. Applications running on Iceberg would have
to map these types onto Iceberg types. Such applications could, in response to
new published version of LDMs, issue ALTER TABLE statements where needed on the
underlying platform, allowing teams to manage schemas of data stores on any
type of platform without requiring personal write access to production
environments.  An important trade off is that making applications interact with
a self-managed catalog of LDMs requires in-house software development. To ease
this somewhat, SDKs for various programming languages can be developed to, for
example, deserialize the schema for the latest version of an LDM based on the
first version and all the modificatoins made to it in later versions.

