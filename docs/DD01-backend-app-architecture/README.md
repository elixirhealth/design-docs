# DD01 - Initial Backend Application Architecture

#### Goals

- describe basic frontend request types sufficient for MVP
- describe high-level backend application architecture sufficient for initial MVP

#### Non-goals

- detail APIs for each backend service
- describe how infra/configs of backend will be implemented
- describe front-end design

#### Key Terms

- an Elxir *user* is the fundamental unit of login
	- may have read/write access to one or more entities
		- for example, a patient may have access to both their own patient entity and also their child's
- an Elxir *entity* is either a patient or physician (office)
	- has a number of public keys it uses to send and receive Envelopes
- a Libri *author* is the entity that sents an Envelope to a *reader*
- Libri document types
	- a [Document](https://github.com/drausin/libri/blob/develop/libri/librarian/api/documents.proto) is the unit of upload/download in Elxir, it is either an Entry, an Envelope, or a Page
	- an [Entry](https://github.com/drausin/libri/blob/develop/libri/librarian/api/documents.proto#L46) contains the content of a document, either as a single Page (if size is less than 2MB) or a set of PageKeys to separate Page documents
	- an [Envelope](https://github.com/drausin/libri/blob/develop/libri/librarian/api/documents.proto#L26) share an Entry's symmetric encryption key with a given reader
		- contains the author and reader public keys
		- contains the encrypted symmetric Entry encryption key
	- a [Page](https://github.com/drausin/libri/blob/develop/libri/librarian/api/documents.proto#L154) is a chunk of a document, limited to 2MB
- a Libri [Publication](https://github.com/drausin/libri/blob/develop/libri/librarian/api/librarian.proto#L195) is created by the Libri Librarian peers that store an Envelope
	- each Librarian subscribes to the publications of other Librarians
	- when a Librarian receives a new publication it hasn't seen before, it sends it to the Librarians subscribed to it
	- these publications are how new Envelope storage events are gossiped around the Libri network


#### Basic Frontend Request Types

The MVP frontend will need to make handful of different requests.
- user focused
	- authenticate a user
	- create/update a user's contact and client device info
	- refresh a user's timeline
- entity focused
	- entity CRUD and lookup
	- search for an entity by some of its characteristics (e.g., name, medical record number, DOB, etc)
	- store entity's encrypted private keys
	- get one or more entity B public keys for entity A to use in sharing a document with entity B
- document focused
	- get the keys of documents sent from/to a given entity within a given date range
	- get one or more documents by their keys

The three categories of frontend requests imply three separate domains for backend services to be ordered under:
- users
- entities
- documents

#### Backend Services

All frontend requests will initially go to the Circulation aggregation/routing service. The service will be very thin, containing no business logic. It's entire job is to route frontend requests to the appropriate backend service(s). 

The Users domain will contain two services:
- *Identity* manages user data
	- endpoints
		- lookup and CRUD name, contact info, client devices
		- lookup and CRUD user -> entity relationships
		- authenticate user
			- most likely will delegate to a 3rd party like [Auth0](https://auth0.com)
	- storage: (GCP managed) Postgres
		- number of users initially will be reasonably small (i.e., less than 10M)
		- strong consistency important
		- DB triggers make auditing changes easy
		- inremental backups easy
- *Timeline* collects document keys that should appear in a user's timeline
	- endpoints
		- get updates within a given date window
	- storage: none
		- service is stateless, collecting data from Catalog and Access services described below

The Entities domain will contain two services:
- *Directory* manages entity data
	- endpoints
		- entity CRUD and lookup
		- entity search
		- entity encrypted keychains (i.e., set of private keys)
	- storage: (GCP managed) Postgres
		- number of users initially will be reasonably small (i.e., less than 10M)
		- strong consistency
		- prefix and full-text search capabilities sufficient for initial search use cases
		- DB triggers make auditing changes easy
		- might/prob want to be able to do joins
		- incremental backups are easy
- *Access* manages entity public keys
	- endpoints
		- add, delete, sample entity author and reader public keys
	- storage: GCP DataStore
		- simple single-table schema; no need for joins or aggregations
			- queries will be simple lists by entity_id
		- Puts/Gets very fast, good since every document share will require a Get
		- sufficient to backup incrementally by created date

The Documents domain will contain two backend services plus the Elxir's Libri Librarians:
- *Librarians* are Elxir's nodes in the Libri cluster
- *Courier* is the interface between Libri and the rest of the Elxir backend
	- endpoints
		- put/get document into/from cache
			- puts don't go to Libri, just to cache and Libri put queue
	- background processes
		- Puts documents from put queue into Libri
		- subscribe to Librarians, sending new Publications to the Catalog
	- storage: GCP DataStore
		- Puts/Gets very fast, likely 2-5x faster than Libri Puts and Gets
		- no need for joins or aggregations
		- can scale to large size (e.g., hold all docs published in last week)
		- no backup needed since it is a cache
- Catalog maintains a durable record of every Libri publication
	- endpoints
		- put/get publications
		- search publications by author or reader public key
	- storage: GCP DataStore
		- fast Puts/Gets/Searches allow it to handle high publication load
		- no need for joins or aggregations
		- can scale to large size
		- sufficient to backup incrementally by created date


Below is a diagram of these services and their interactions.

```
        ┌─────────┐                                               ┌─────────┐
        │3rd party│                                               │ client  │
        └─────────┘                                               └─────────┘
             ▲                                                         │
             │                                                         │
             ▼                                                         │
  ┌─────────────────────┐                                              │
  │        Libri        │                                              │
  └─────────────────────┘                                              ▼
             ▲                                                       ┌───┐                                  public internet
─────────────┼───────────────────────────────────────────────────────┤ELB├──────────────────────────────────────────────────
             │                                                       └───┘                        private application layer
             │                                                         │
             │                                                         ▼
             │                                                  ┌─────────────┐
             │                                                  │ Circulation │
             │                                                  └─────────────┘
  ┌ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─                │
             ▼                                         │               │
  │   ┌─────────────┐                                                  │
      │┌────────────┴┐      ┌──────────────────────────┼───────────┬───┴────┬────────────────────┬───────────────┐
  │   └┤┌────────────┴┐     │                                      │        │                    │               │
       └┤ Librarians  │     │                          │ ┌ ─ ─ ─ ─ ┼ ─ ─ ─ ─│─ ─ ─ ─ ┐ ┌ ─ ─ ─ ─ ┼ ─ ─ ─ ─ ─ ─ ─ ┼ ─ ─ ─ ─
  │     └─────┬───────┘     ▼                                      ▼        │                    ▼               ▼        │
              │      ┌─────────────┐   ┌─────────────┐ │ │  ┌─────────────┐ │        │ │  ┌─────────────┐ ┌─────────────┐
  │           └─────▶│   Courier   │──▶│   Catalog   │◀─────│  Timeline   │─┼────────────▶│   Access    │ │  Directory  │ │
                     └─────────────┘   └─────────────┘ │ │  └─────────────┘ │        │ │  └─────────────┘ └─────────────┘
  │                         │                 │                             │                    │               │        │
                            ▼                 ▼        │ │                  ▼        │ │         ▼               ▼
  │                  ╔═════════════╗   ╔═════════════╗               ┌─────────────┐      ╔═════════════╗ ╔═════════════╗ │
                     ║  DataStore  ║   ║  DataStore  ║ │ │           │  Identity   │ │ │  ║  DataStore  ║ ║  Postgres   ║
  │                  ╚═════════════╝   ╚═════════════╝               └─────────────┘      ╚═════════════╝ ╚═════════════╝ │
   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘ │                  │        │ └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                        Documents                                           ▼                        Entities
                                                         │           ╔═════════════╗ │
                                                                     ║  Postgres   ║
                                                         │           ╚═════════════╝ │
                                                          ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
                                                                     Users
```












