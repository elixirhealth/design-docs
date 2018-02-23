# DD03 - Directory Service

This design doc describes the Directory service implmentation details, including API, storage, search, and roadmap.

#### Goals

Describe
- entities
- API
- storage implementation details
- search queries and ranking
- roadmap

#### Non-Goals

Describe
- access controls (i.e., who can search for and find what)
- monitoring & alerts
- entity type specifics (e.g., geocoding addresses)

#### Entities

Entities can be patients, provider offices, providers, organizations, or any other type of actor in our healthcare system. Entities share a common general structure:
- entity ID
- type attributes
- organization identifiers

The entity ID is a random base-32 string with a single-character type prefix (e.g., `P` for patient) and a [Luhn checksum](https://en.wikipedia.org/wiki/Luhn_mod_N_algorithm) character suffix to detect simple typos. The total ID length is initially 9 characters, which gives 7 characters of entropy (minus the type prefix and checksum suffix), or `32^7 = 34B` possibly IDs for each type. The large size of this ID space means that we don't have to worry too much about the [birthday problem](https://en.wikipedia.org/wiki/Birthday_problem) and can just retry with a new ID in the very rare case of generating a duplicate ID. This ID format should make transcribing it in person (or over the phone) simpler and safer than if we were using a large number or UUID.

The attributes for a given type are structured and well defined. For example, a `patient` will always have a first & last name and birthdate, and an office will always have a name and address. While some empty values are allows (e.g., name suffix), we assume that these structures attributes will be fairly densely populated. Each type will have its own set of attributes, which may grow (hopefully slowly and additively) over time.

We will also want to identify entities by identifiers given by other origanizations, for example
- patients may have medical record numbers (MRNs) for each provider organization
- doctors and facilities will have national provider identification numbers (NPIs)

These identifiers will generally be sparse (a patient will probably have only a handful of MRNs and certain organizations), so it makes sense to store them separately from the dense type attributes. These identifiers will have the following structured attributes:
- identifier type
- organization entity ID
- identifier value
Each entity may have a list of these organization identifiers.


#### API

The Directory (grpc) API will define basic operations on entities.

Endpoints
- `PutEntity` adds a new or modifies an existing entity
- `GetEntity` returns an existing entity
- `SearchEntity` returns a list of entities matching a query
- `PutEntityIdentifier` adds a new or modifies an existing organization identifier
- `SearchEntityIdentifier` returns a list of entities matching a given identifier query

We will probably want to add corresponding `Delete` endpoints as well, but for now we will delay defining them until the use case is more clear.


#### Storage options

Postgres will provider durable storage for the service, though we will also implement an in-memory storer for use in testing. When considering storage options, we have the following desiderata:
- needs to be able to back multiple service replicas
- all queries should be relatively fast (less than 50ms)
- needs to support search of strings using more than just equality filters
- should at least have a highly available read path (i.e., gets and searches)
- should allow auditing and backups with relative ease

We considered a few main options:
- GCP DataStore only
- GCP DataStore + Elastic Search
- Postgres only
- Postgres + Elastic Search

GCP DataStore (DS) is attractive because it offers very fast Puts/Gets along with structured attributes and high availability for both writes and reads, but queries can only involve equality and a single inequality filter, so it wouldn't be able to handle more complex search queries. Also, the history auditing and incremental backup story for DataStore is not great, so we'd have to implement much of that ourselves.

If we added Elastic Search (ES) to DS, we would get ES's very powerful indexing and ranking capabilities along with its high availability. But adding ES means that now our entity data lives in two places that need to be kept in sync, somethign that's certainly possibly but more hassle than just having a single source of truth to handle all queries. And ES doesn't amerliorate any of the auditing or backup manual work we still would have to do with DS. Finally, we would have to host and managed ES ourselves, which of course is doable, but more of a pain than using Google-managed infra like DS and Postgres.

The Postgres only option is attractive for a number of reasons: 
- indices will allow single-entity Puts/Gets to be relatively fast (though not as fast as sub-10ms DS)
- B-tree indices allows for fast equality and prefix (anchored) searches, and GIN trigram indices allow for fast, unanchored searches, robust to simple mispellings
- can deploy a number of read replicas for read-heavy load
	- write redundancy will just be a hot standby
- triggers can automatically write to history tables when entities are updated or deleted
- incremental backups are a thing
Its drawbacks are:
- writes not highly available
- Put/Get queries prob not as fast as those to DS, Search queries prob not as fast as those to ES
- will need to implement some of search result ranking in code (rather than delegating to ES)

At some level of scale, it may/probably will become necessary to shift the read load from Postgres to ES, allowing us to keep many of core Postgres benefits. But we can probably handle this scale in the intermediate term with many, large read replicas, so it doesn't make sense manage ES infra and keeping two data sources in sync at the momment.


#### Postgres DB implementation details

The DB will have one schema:
- `entity` contains entity attributes

The `entity` schema will contain a separate table for each entity type's structured attributes. For example:
- `patient`
	- `last_name`
	- `first_name`
	- `middle_name`
	- `suffix`
	- `birthdate`
- `office`
	- `name`
	- `location`
- `organization`
	- `name`

Each table will have the following rough structure:
- `row_id` internal autoincrementing ID
- `entity_id` not-null unique varchar
- `transaction_period` not-null tstzrange of the form [created_time, deleted_time)
	- in the "current" table, the deleted_time value will always be `NULL`, but in the history table, it will be set
- (nullable type attribute columns...)

All type attribute columns (e.g., patient name, birthdate, etc) will be nullable to keep business logic out of the DB. (In general, storage should be as ignorant as possible about business logic. Rules about what attributes must be populated (e.g., last name) should reside in the application layer. Keeping the storage layer simpler also means that changes in business logic can mostly happen in one place (the application code) rather than two (code & DB).)

Organization identifiers will be stored in separate `[type]_identifier` tables, so for the 3 entity types above,
- `patient_identifier`
- `office_identifier`
- `organization_identifier`

each with the following schema:
- `row_id` internal autoincrementing ID
- `transaction_period` not-null tstzrange of the form [created_time, deleted_time)
- `identifier_id` not-null unique varchar
- `entity_id` not-null varchar, foreign-key'd to `entity_id` in corresponding type table
- `organization_entity_id` not-null varchar foreign-key'd to `entity_id` in `organization` table
- `identifier_type` not-null varchar (probably from application-level enum)
- `identifier_value` not-null varchar

Each table described above will have a corresponding history table that tracks the value of each row during some transaction period in the past.
- `patient_history`
- `patient_identifier_history`
- `office_history`
- `office_identifier_history`
- `organization_history`
- `organization_identifier_history`

The operations required to populate the history tables are managed by DB triggers:
- on row update
	- insert a new row into history table with `deleted_time` bound of `transaction_period` populated withe current timestamp
	- delete existing row in current table
	- insert new row (i.e., new row_id) with updated value(s) into current table
- on row delete
	- insert a new row into history table with `deleted_time` bound of `transaction_period` populated withe current timestamp
	- delete existing row in current table

Entity search will require the appropriate indices to make these operations efficient. "Search" encompases both auto-complete and full-search uses cases, where the former must return a few results very quickly on usually incomplete queries, and the later can be a bit slower but should be more comprehensive. In both cases, the search request will define a limit, indicating the maximum number of results to return. Search will not support paging because we believe that if the results are not found on the first page, the query should be refined.

For the three entity types defined above, we will initially support searches on the following fields (or combinations of fields) via the given indices
- `patient`
	- `entity_id` via B-tree index to support prefix and equality matching
	- `last_name` + `first_name` via trigram index to support mispellings and different name orders
- `patient_identifier`
	- `identifier_value` via B-tree index to support prefix and equality machine
- `office`
	- `entity_id` via B-tree index to support prefix and equality matching
	- `name` via trigram index to support mispellings and partial, unanchored (i.e., in the middle) substrings matches
	- address string (form still TBD) via trigram to support mispellings and partial, unanchored substring matches
- `organization`
	- `entity_id` via B-tree index to support prefix and equality matching
	- `name` via trigram index to support mispellings and partial, unanchored (i.e., in the middle) substrings matches

Each field (or fields) searched also returns a similarity score in [0, 1], where 1 indicates perfect similarity, and 0 perfect dissimilarity. Each index type implies its own way to compute the similarity between a `query` and an existing indexed `value`:
- B-tree
	- value: just the indexed field value
	- predicate: `'query%' LIKE value` performs a prefix (including equality) match
	- match similarity: `char_length(query)::real / char_length(value)::real`
		- the index and predicate imply that when the query matches the value, it is always a prefix of (or equality to) it
- GIN [trigram](https://www.postgresql.org/docs/current/static/pgtrgm.html)
	- value: the (possibly composite) indexed field values with NULLs replaced by empty strings
		- e.g., `UPPER(COALESCE(first_name,'')) || ' ' || UPPER(COALESCE(last_name,''))`
	- predicate: `query % value`
		- matches are defined by a similarity greater than then similarity threshold (default 0.3, but prob will need to be adjusted)
	- match similarity: `similarity(query, value)` is the [Jaccard index](https://en.wikipedia.org/wiki/Jaccard_index) between the two sets of trigrams

When matches are found in each query, the match similarity is also returned for use in match result merging (see below).

Database DDL migrations will be managed by the [github.com/mattes/migrate](https://github.com/mattes/migrate) package with DDL SQL files managed by [github.com/jteeuwen/go-bindata](https://github.com/jteeuwen/go-bindata). When a Directory service starts up, it migrates the DB up to the latest version, if necessary. Some tests maybe also spin up and migrate a test DB within the build container to integration-test things that are difficult to mock.


#### Search queries and ranking

The simplest search is just a single query string, akin to the Google search bar. The search finds all possible locations in parallel for (some or all) of that string and collects the results into the response. The search request also includes a limit on the number of results.

Search filters allow the user to narrow the search when the client has a specific use case (e.g., looking up a patient by the medical record number). For now, they are just boolean indicators for the types to search and are by default `true`.
- `include_patient`
- `include_patient_identifier`
- `include_office`
- `include_organization`

When matches from more than would source are found, we need to combine their results into a merged, ordered result set. We expect most of the queries to be primarily targetted to a single field or sets of field, so matches in more than one source are probably infrequent. But queries like "NYC Sports Medicine Union Sq" contain both an office name ("NYC Sport Medicine) as well as part of the location ("Union Square"), and we want the appropriate office to be ranked higher than all other "NYC Sports Medicine" named offices or those in "Union Square".

Elastic Search has sophisticated parameters for tuning the equation that combines multiple similarities scores into a single one. Since we're not using Elastic Search, we need to define our own approach. If `s` represents a vector of similarities, each in [0, 1], then we will define our aggregate similarity as the L2 [norm](https://en.wikipedia.org/wiki/Norm_(mathematics)) of s, i.e., the square root of the sum of squares. The L1 (i.e., sum) or even LInf (i.e., max) norms might be more appropriate in some use cases, but we'll start with L2 because it works pretty well on a lot of applications.

#### Roadmap

There's clearly a lot here. The Directory service will probably be the most complex initial backend service thanks to the search capability. Fortunately, the design above doesn't have to be fully implemented before it can be used. One possible feature roadmap is
- implement `PutEntity` and `GetEntity` endpoints only for `patient` and `office` entity types
	- omit `office` location attribute, since it will be a little complicated/TBD with geocoding, etc
	- omit DB history triggers or tables
	- omit organization identifiers from API and storage
- implement `SearchEntity` only for `patient` and `office` entity types
	- with same omissions
- implement in-memory storage backend
- implement rest of service pieces (server, CLI, acceptance tests, etc) --> useable basic service at this point
- add organizations and organization identifiers to API and storage
- add history tables
- add office location attribute(s)










