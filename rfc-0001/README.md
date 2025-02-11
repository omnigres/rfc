# Omnigres Schema Evolution Management Proposal 

Managing schema evolution, especially in presence of data-native functions
(fka _stored procedures_) is a non-trivial task:

* How do we prevent data and auxiliary information (such as presence of indices) loss?
* How do we ensure that schema is evolved as intended?
* How do we make sure the correct set of extensions at correct versions is used for every specific version?
* How do we sure data-native functions correspond the correct revision of the schema?
* and so on.

This proposal captures the approach Omnigres is using. Many parts of it are not implemented yet. The purpose
of the proposal is to capture ideas and findings in a tangible way so that the implementation can be done
expediently.

## Starting point

Core thesis behind this proposal is that in a business system project, schema and code are maintained
cohesively in a single repository. Currently, this is implemented as a _source directory_ that
contains SQL and other files (such as Python modules) that can be provisioned.

Using `omni_schema` extension and its _schema assembly_ functionality, given the _source directory_
it can assemble the schema files and statements in a correct order. This is achieved through brute-forcing:
it runs statements and non-SQL files one by one until an error occurs and attempting to resolve the problem
by running other statements and files until completion or an error loop. This is done in a brand-new database
to avoid any possible interference.

The idea is that engineers maintain the _desired schema_ and code visibly in the _source directory_ and this
directory is the absolute source of truth.

## Evolution begins here

The above schema works if we never need to migrate a (production) system with data in place.
However, this expectation is neither realistic nor practical.

When we are ready to capture a revision of the schema (including the very first one), we should start
revision identification so we can easily refer to them in the future.

Let's use UUID v7 (time-ordered) identifiers. They fit UUID format, they are sorted and convey the (non-strict) notion of time.

Now, in order for revisions to be properly ordered, without relying on time, let's propose that every revision can refer to
zero or more parent revisions. There could be only one revision with zero parents (the very first one) and we should always
make revisions converge to a linear ending by having one "interim" final revision so we don't have much divergence. We'll cover that later.

For better comprehension, let's visualize such revisions as directories "revisions" directory, named after their ID:

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
```

Since we need to store parent references, let's say every revision will have some
kind of metadata files:

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
```

### Metadata

**metadata.yaml**
```yaml
# To make sure we know what format is this
"$schema": "https://schema.omnigr.es/revision/v1.json"
# List of parent IDs this revisions joins
parents: []
```


> [!IMPORTANT]  
> Metadata file should not contain any defaults to avoid misinterpretation
> when the defaults will inadvertently change.

### Sources

Now, since for every revision we should be capturing all the 
source code involved (to avoid any ambiguity about it), let's capture them all 
in a single file (to avoid polluting project repository with multitudes of files
on every revision capture)

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
```

**sources.dat** is a provisional name as there are a few formats worth discussing:

| Format           | Pros                                   | Cons                                                                                  |
|------------------|----------------------------------------|---------------------------------------------------------------------------------------|
| Tar              | Easy to open using command-line tools  | Not directly understood by Postgres unless copied through a program; uncompressed     |
| Tar.(gz/bz2/...) | Same as tar + Smaller size             | Same as tar                                                                           |
| Zip              | Same as tar + Smaller size             | Same as tar                                                                           |
| SQL              | Can be loaded by sending into Postgres | Can modify arbitrary objects, not directly usable in command-line for file extraction |
| text/csv/json    | Can be loaded using COPY in Postgres   | Not directly usable in command-line for file extraction                               |

Other suggestions are welcome.

### What changed?

Here's the next piece of upcoming functionality in omni_schema – [schema diffing](https://github.com/omnigres/omnigres/pull/766). In a nutshell,
what it allows to do is compare two schemas represented in a highly normalized fashion, so we can compare which relations were added and which
were gone. 

High-level (without getting into precision of how they are currently represented in omni_schema, though
the departure is now significant from the example), something like this could happen when a table was renamed and a
column was added (omitting some for brevity):

```
removed_table(emp)
removed_column(emp, id)
removed_column_type(emp, id, int)
removed_column(emp, email)
removed_column_type(emp, email, text)
added_table(employe)
added_column(employee, id)
added_column_type(employee, int)
added_column(employee, email)
added_column_type(employee, email, text)
added_column(employee, name)
added_column_type(employee, name, text)
```

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
```

**diff.yml**

```yaml
- removed_table: # ...
- added_table: # ...
# ...
```

This file captures which changes were found between parent revision and the current one. This is important for tracking
purposes (and avoidance of implicitness), but also lets us get to a very interesting point:

However, when do a migration for a revision, we now have a precisely defined "change" test that we can use to make sure that the
migration script has led to the original desired schema.

#### Re-interpreting diffs

In the above example, we looked at the representation of renaming, but we capture just the changes. It is possible
to define a set of heuristics that can offer an "explanation" of the underlying change, compressing the above to:

```
renamed_table(emp, employee)
added_column(employee, name)
added_column_type(employee, name, text)
```

Such heuristics should ideally be done automatically and presented to the developer and its agents
to ensure their accuracy.

The fidelity of testing is even better. For example, we can test more precisely by ensuring the original data 
is intact or checking that the OID of the table has not changed (though this is a weak test as migration methods
may call for re-creating a table.)

### Migration

A migration is something that can take us from a set of parent schemas and bring us to the desired schema. Traditionally,
this has been often done by some form of migration scripts, either pure SQL (or PL/pgSQL) or some kind of DSL.

So, let's start at the very basics – SQL (and PL/SQL).

We can add either a single file:

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  migrate.sql 
```

Or a directory of (lexicographically ordered?) files:

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  migrate/
    1_types.sql
    2_tables.sql
```

For completeness, we might want to be able to pursue reversal of the schema:

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  undo.sql 
```

#### Transformation functions

The SQL migration might need to do data transformation. It can do so in-band,
but it might be useful to add convention for defining SQL or PL/pgSQL functions
in a specific place so they are easy to locate:

```shell
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  migrate.sql 
  transformations/
```

The exact convention/layout of files and function names can be discussed later,
as it may be important.

#### Migration DSL

Migration language expressed in SQL (or even PL/SQL) suffers from a few problems:

1. It captures what to do, not a high level definition of happened.
2. Specifies a particular way of doing migration – and might not be suitable in some cases. For example, if migrating a
   minute amount of data, a simple update is fine, but if the relation is large, a batched approach is better.
3. Better implementations may become available (or known at a later time)

We might want to propose using a sort of declarative language that would capture the essence of changes better. Most of the operations can be expressed
in a form of "morphism" primitive. 

A set of relations, and a set of columns in each, "consumes" them and produces a set of relations 
with a set of columns (the relations do not need to be new):

```
[emp(id, email)] => [employee(id, email, name)]
```

or

```
[emp(id, email)] => [employee(id, email)]
[] => [employee(name)]
```

(the notation is omitting details for brevity)

This can be further lifted into **split/merge** morphism, where one is taking relations
and splits the columns into other relations. Merge does the reverse. Going one step further,
some of these can be specialized into **normalize/denormalize** morphisms which capture the essence
of these common operations even more accurately by identifying their purpose.

So, if we design such a small language and augment it with transformation functions (which will
now be easier to design a convention for given the morphism structure), we can replace `migration/undo.sql`
with just one such definition.

However, by itself, it won't be able to enact the migration. For these purposes, `omni_schema` would
need to be able to do so. The really interesting benefit of such DSL is that we can now enact the migration
in different ways:

* We can naively translate it to SQL statements. They are going to be of a destructive/interruptive nature,
  but they will do the job as most migrations do.
* We can apply rolling updates in schemas, similar to [pgroll](https://pgroll.com/)

### What about data?

Some data in schemas is structural to the application. For example, currencies used in the system or
types of accounts. We need to be able to migrate those into staging and production, too.

To this end, this proposal recommends observing if any of the tracked relations contain any data and
capturing them in data files (text/csv/json?)

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  migrate.sql 
  transformations/
  data/currency.csv
  data/account_types.csv
```

In most cases, we just want to ensure that records in those relations make it into migrated schema (and we can
use this capture as a test).

However, sometimes we need to stipulate that some data must be gone. We can perhaps use some metadata to specify
if data in some of the tables is meant to be present to the exclusion of any other data.

```
0194f2ae-5a6f-762a-8ec4-d06d609f14fc/
  metadata.yml
  sources.dat
  PARENT_ID.diff.yml
  migrate.sql 
  transformations/
  data/currency.csv
  data/account_types.csv
  data/metadata.yml
```

***data/metadata.yml**:

```yaml
"$schema": "https://schema.omnigr.es/revision/data/v1.json"
currency: present
account_types: only
```

Further elaboration on these profiles of data presence is needed.

### Effect changes

We have a case where loaded Python files provision data-native functions. From the perspective of schema
diffing, they will look like changes to plpython function with all the wrapping generated by `omni_python`. 

Should we capture the effect-of relationship here? Should we only track what actually changed – the final
function's code, the effect-of relationship or both?

## Benefits of the proposal

* Highly explicit
* High degree of testability
* Ability to quickly provision any revision without remigrating due to the full source directory capture

## Drawbacks

* A somewhat new workflow
* Bigger repository footprint