# Timeline service

Timeline service returns a timeline of documents sent and received for a given user.

#### Goals

- motivation
- API


#### Non-Goals

- monitoring & alerts


#### Motivation

The timeline is the user's main landing spot in the front end. It contains summaries of the documents they uploaded and those shared with them. The frontend queries from a particular point in time, and the timeline returns a given number of document summaries up until that point in time. These documents are a combination of Libri Entry summaries and Envelope sharer identification, so the frontend will look something like

- Local Imaging Office shared L5-S1 MRI with you. (7/1/2016)
- You shared back pain summary with Dr. James McAvoy (6/29/2016)
- You uploaded back pain summary. (6/28/2016)
- Dr. Jie shared visit summary with you. (6/25/2016)

Each timeline element of the response
- received_time
- author_entity
	- entity ID
	- name
- reader_entity
	- entity ID
	- name
- envelope
	- fields...
- entry_subset
	- created_time
	- metadata_ciphertext
	- metadata_ciphertext_mac


#### Dependencies

The timeline service won't have any state of its own. Instead, it will integrate state from most of the other Elxir services:
- get entities associated with user_id from User service
- get pubkeys associated with these entities from Key service
- get publications to these reader pubkeys from Catalog service
- get entities associated with these author pubkeys from Key service
- get entity details associated with these author entities from Directory service
- get envelopes of from these publications from Courier service
- get entry metadata of from these publications from Courier service


##### API

For now, the service will just have a single endpoint, `GetTimeline` that returns up to a certain number of events before a given time.


