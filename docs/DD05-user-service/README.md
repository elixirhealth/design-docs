# User service

This DD describes the planned User service, which manages attributes of Elxir users, including which entities they are associated with. We expect to add more features as they become necessary.


#### Goals

- motivation
- auth0
- initial API
- storage


#### Non-Goals

- user authentication & authorization
- monitoring & alerts


#### Motivation

Elxir users are the units of login. Each user is associated with one or more entities, which are the units of data. For example, user john@doe.com might be associated with his own "John Doe" entity as well as "Jim Doe," his infant son, whose medical records he manages. Similarly, user alice@bates.com might have her own "Alice Bates" entity and also be associated with "Local Imaging Office" entity. Users can be associated with multiple entities, and an entity may be associated with multiple users. One of the main roles of this service is to track the user-entity relationships.

This service will also own other information about users: contact info (email and phone number), preferences, connected devices, IP addresses, etc.


#### auth0

[Auth0](https://auth0.com) is a 3rd party identity provider that we plan to use for managing most of the simple aspects of a user's identity, including login (passwords and other 3rd party identities), email verification, multi-factor authentication, and other details common to most apps. Auth0 has a number of features that make it attractive:
- simple [Lock](https://auth0.com/docs/libraries/lock/v11) widget and library for Elxir frontend to talk directly to Auth0 (meaning sensitive passwords never hit Elxir's backend infra)
	- also has good libs for native apps as well
- authorization management between users and specific public-facing Elxir endpoints (in Circulation service)
- a comprehensive [management API](https://auth0.com/docs/api/management/v2) that the Elxir backend can use for managing user details
- a nice [user dashboard](https://auth0.com/docs/dashboard) for human interaction
- logical separation of tenants (e.g., prod vs. staging)
- support for Enterprise identity providers (e.g., ldap, AD, etc)
- strong auditing and HIPAA compliance
- free tier
- 99.95% uptime SLA

In addition to identity, Auth0 has the ability to also store some limited additional info in the `user_metadata` field. We probably want to use this sparingly, if at all, and instead store that info in Elxir's own User service storage.


#### initial API

Since Auth0 will handle login and storing basic things like email, devices, etc, the User service will manage additional attributes associated with each user. The list of these things will grow with Elxir's features, but for the MVP, it will just involve user-entity relationships. 

- `GetEntities` returns the active entity IDs associated with a given user ID
- `AddEntity` associates a given entity ID with a given user ID

Endpoints we imagine adding later on include
- `RemoveEntity` removes an entity ID associated with a given user ID
- `UpdateEntity` updates the entity-user association
	- relationship: self, parent, employer, etc
	- role: admin, author, reader
- `GetUsersByEntity` returns the users associated with a given entity
- `PutPreferences` create or update user preferences
- `GetPreferences` gets the user preferences


#### storage

The user data stored in Elxir will generally be pretty small, but some of it (e.g., user-entity associations) will need to be queryable very quickly (ideally in less than 10ms), and will need to scale horizontally with the number of users. These latency and scaling considerations incline us toward using GCP DataStore for storing this information.

We will start with just a single `user_entity` kind with the following fields
- `user_id` is the user string ID
- `entity_id` is the entity string ID
- `removed` is a boolean flag indicating whether the relationship has been removed (initially `false` for everything until we add the `RemoveEntity` endpoint)

It will also have the following audit fields:
- `added_time` is when the key was added
- `disabled_time` is when the key was disabled
- `modified_time` is the `added_time` if `disabled_time` is empty, otherwise it is `disabled_time`
- `modified_date` is the epoch date (days since epoch)
	- will mostly be useful down the road for filtering rows for incremental backups


#### roadmap

- DataStore storer handles Adds and Gets for user-entity associations
- flesh out service code for `GetEntities` and `AddEntity` endpoints
- add memory storer
- add acceptance tests
- add CLI and local demo --> MVP service at this point
- add other endpoints as needed
	- some may need to hit Auth0's [management API](https://auth0.com/docs/api/management/v2)




