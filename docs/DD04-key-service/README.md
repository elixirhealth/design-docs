# Key service

This design doc describes the the Key service, which manages entity public keys. This doc covers implementation details, including API, storage, and roadmap.

#### Goals

- motivation
- API
- storage implementation details
- key sampling

#### Non-Goals

- encrypted keychain storage
- access controls (i.e., what entity A keys entity B can sample)
- monitoring & alerts

#### Motivation

Elxir entities use assymetric elliptic curve key-pairs to securely share data with each other. If Entity A wants to send data to entity B, they need one of entity B's public keys, which, when combined with one of A's private keys, forms a shared secret between A and B that each party can decrypt with their private key and the other's public key. If B sees that someone with public key X has sent them some data (using B's public key Y), B would want to know which entity owns public key X.

Public keys are represented as byte arrays. Different elliptic curves and key encodings will produce different size byte arrays, but we assume that the public keys are secp256k1 curve keys, represented in compressed (33-byte length) form, as is used by the Libri network. Entities interacting with Libri have two sets of keys, author key and reader keys. Author keys are used to send documents, and reader keys are used to receive them. The Key service will distinguish between these two types of keys, though everything else about them is the same.

The Key service manages the public keys associated with each entity. Each entity securely stores the corresponding private keys on their client, so Elxir never has access to these. The main functions of the service are:
- add a new public key for a given entity
- get the details (including ID of entity owner) of public key
- disable an existing public key
- sample a set of entity A's public keys for entity B to use to send data to A

This service will allow the following user-focused actions:
- adding public keys from a user's new keychain
- rotating an entity's public keys available for sampling
- getting the entities who have sent a user documents (via Libri Envelopes)
- getting a set of entity A's public keys for entity B to use in sending documents to A

As far as this service is concered, any entity B can get a sample of entity A's public keys. In the future, we may want to limit the discoverability of entity A by entity B via re-identification. If/when we want to add such limits, they should exist in a separate service. We discuss of B's sampling of A's public keys below.

#### API

Rather than working with individual public keys, the API defines most actions on sets of keys, since the use cases often involve workign 

Endpoints:
- `AddPublicKeys` add a set of public keys associated with a given entity ID
- `GetPublicKeys` returns the full key info (including entity ID) given one or more public keys
- `DisablePublicKeys` disables one or more public keys for sampling
- `SamplePublicKeys` returns a sample of entity B's public keys for entity A to use

In order to preserve the historical record, public keys can only be added and disabled, but not deleted.

#### Storage implementation details

DataStore will provide highly-available, fast storage for the service, though we will also implement an in-memory storer for use in testing. The main factors in considering storage options were:
- storage queries only need simple equality filters (i.e., nothing fancy) and CRUD operations (i.e., no joins or aggregations)
- `GetPublicKeys` and `SamplePublicKeys` will need to be very fast, since they will be called for every document published
	- may later add client-side public key caching (which will probably be good to have down the road), which will reduce load on `SamplePublicKeys` a bit, not not on `GetPublicKeys`
- information associated with each key is pretty simple and is represented well by one table
	- no need for strong consistency, really
- should be able to handle O(100M) to O(1B) public keys in order to support O(1M) to O(10M) users
- should have reasonable backup story

Since the keys will be stored in just a single table with basic filtering and CRUD operations, DataStore's API should suffice, and it's high-availability and fast read/write performance should allow it to handle an increasing user load better than Postgres. Since DataStore is fully managed instead of partially managed (like Postgres via Google Cloud SQL), it should have slightly lower operational load. While we have seen reasonable Postgres performance up to 1B rows in a table, if we have strong user growth over first few years, we will hit that barrier.

The most compelling reason to consider Postgres here is its superior incremental backup capability, which is fairly turnkey. DataStore support bulk export and import operations, but they are eventually consistent and dump/reload everything in a table (i.e., not incremental). This bulk export means that backups more frequent than once a day are probably not reasonable, both for performance and cost. An alternative would be to write out own incremental backup utility to query on a low-cardinality, monotically-increasing value (like date a row was modified). Restore operations would still involve loading everything back into the instance, but hopefully that would be quite rare.

We think it's reasonable to assume that the combination of a very simple data schema, fully-managed infra (so no upgrades, etc), and bulk export/import mean that DataStore will be safe enough for a while to use until we can implement our own incremental backup solution for ita.

Public keys will be represented in DataStore in the `public_key` kind with the following fields
- `public_key` is the (66 char) hex of the 33-byte secp256k1 public key
	- this is the key of the table
- `entity_id` is the (usually 9-char) ID of the entity that owns the public key
- `key_type` is either 'AUTHOR' or 'READER'
- `disabled` is a boolean flag indicating whether the public key has been disabled (and thus should not be sample-able)

It will also have the following audit fields:
- `added_time` is the epoch timestamp (in micros) of when the key was added
- `disabled_time` is the epoch time (micros) of when the key was disabled
- `modified_time` is the `added_time` if `disabled_time` is empty, otherwise it is `disabled_time`
- `modified_date` is the epoch date (days since epoch)
	- will mostly be useful down the road for filtering rows for incremental backups

We will make use of DataStore's `GetMulti` and `PutMulti` operations to make getting and adding/updating sets of keys efficient.

#### key sampling

Multiple keys per entity give anonymmity when every Libri data exchange is public. Knowing that author pub key X sent something to reader pub key Y does not tell you who is sending data to whom, unless those entities publish their public keys somewhere else. But if every entity just used one public key, it might be possible to re-identify entities based solely on who is sharing data with whom. To thwart any malicious activity like this, each entity has many public keys, say 64 author and 64 reader, so reidentification becomes practically infeasible. 

When an entity A wants to send data to entity B, A requests a sample of B's keys (say, 8) and then randomly selects a key from that sample to use for each exchange. A malicious client intent on discovering all of B's reader public keys (so as to re-identify all their data exchanges) might request many samples and with enough requests indeed get every one. To thwart this possibility, A only ever sees a certain number (say, 8) of B's keys. This number is thus the maximum number of keys that A can sample for B. If B rotates their keys, adding some new ones and disabling others, A will get some/all different keys after this rotation, but it will still always be a subset of B's full key-set.

One approach to implementing these sampling limitations would be to split each entity's keys up into batches of 8, say, and store which batch every other entity has access to. But storing and updating that access is a bit annoying, so we'd prefer to avoid it if we can. Instead, we propose the following when A requests a sample of B's keys:
- service get's all of B's active reader keys from storage (e.g., 64 keys)
- service then computes the the SHA256-HMAC of each key using B's entity ID as the MAC key
- keys are ordered by these MACs and the first 8 are available for sampling

This approach does not require storing ACLs between pairs of entities and also evolves easily as new keys are added and old keys are disabled. It also guarantees that for a fixed set of B's keys, A always samples from the same subset of 8. 


