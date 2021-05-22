---
layout: post
title: "Vartalap: Chat Server Architecture V2"
date: 2021-05-22T11:53:47.098Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This is third blog post in series of blogs posts, where I will be sharing details about Vartalap. How it started, decision I make while development, technical details, problem with current system and how I am planning to solve it. In this blog I will be sharing the new backend server architecture, I come up with to overcome the existing limitations. If you haven't already read the first and second blog, I highly recommend reading them. 
Links to previous blog posts:

* [Vartalap: Open Source Personal Messaging App](https://blog.one9x.org/vartalap/2021/04/04/vartalap-personal-messaging-app.html).
* [Vartalap: Chat Server Architecture](https://blog.one9x.org/vartalap/2021/04/10/vartalap-chat-server-architecture.html).

### Quick Recap: Limitation of Current Architecture

* No proper acknowledgement system for message delivery.
* Only web socket gateway for events, making it difficult for background event syncing.
* Router publishing directly into gateway topic, making message delivery logic to be duplicated among services.
* Group operations are synchronous making it not useable offline.
* Base message template for events.

# Architecture V2:

New Architecture update:

![Vartalap Architecture V2](images/uploads/architecture-v2.png "Vartalap Architecture V2")

In V2 version of the system external interface of the system remains the same, however there are new components are introduced and the responsibilities has been distributed/ transferred within the components. 

### New Components:

We have two new services

* **Rest Gateway (HTTP)**

  * Provides the rest endpoint `/send` to client to send the messages via rest API call instead of just relying on the Web Socket.
* **Persistence Storage Rest Endpoint (HTTP)** 

  * Provide the rest endpoint `/messages` to client to fetch the messages.

### New Responsibility of existing components:

* **Web Socket Gateway**

  * Instead of listening to a `Kafka topic` for messages to send how it will expose an internal endpoint `/send`. Which will be used to push messages to client. 
    Advantage of using Sync communication (endpoint) instead of Async communication (Kafka topic)

    * Real time update on message status, if message is sent to client or not.
    * Better retry logic.
* **Session MS**

  As `Session MS` already has the user connection state, it is an ideal candidate for handling routing message to gateway's instead of just act as store. 

  * Route message to respective gateway, include routing offline messages.
  * Retry failed message.
  * Handle message acknowledgment.

    Instead of using persistence storage service to store every message and delete them when ack is received. It will be more efficient for `session ms` to wait message ack and act accordingly like retrying or treating it as offline message as `session ms` is storing the message in memory store like `Redis` instead of on disk stores.
  * Maintain user connection state.
* **Message Router**

  Message router responsibilities as now slim down, new responsibilities are:

  * Message parsing and decoding.
  * Forwarding message to respective services like group messages to `group message router`, one to one to `session ms`, etc.
* **Group Message Router**

  Group message router now instead of completely relying on `group ms` for getting group data, as they both belongs to same category, it can show access data directly from the database. (Note: access is read only).

  * Message parsing and decoding.

    This includes updating message with users of the group, and decoding message to get relevant info.
  * Forwarding message to respective services like group upsert message to `group ms`,  chat message to `session ms`, etc.
* **Persistence Storage Service**

  * Handle ack message as forwarded by `session ms.`



## Updated Request Flow

#### Send Message (1 to 1)

**Client Side**

* Send message against a chat.
* Store message against chat in local store.
* Publish message to web socket. (if connected).
* Publish message using rest endpoint from background service.
* Send message ack against received message.

**Server Side**

* `Gateway` publish `onMessage` event to message broker to meta info like from.
* `Message router` listen for `onMessage`.
* Parse the message and extract destination information.
* Publish message to `send message` event to message broker.
* `Session MS` listen to `send message`.
* If user is connected.
* It will send message to gateway using `Gateway` send endpoint.
* `Gateway` tries to send message to user socket.
* If failed, return failure response.
* `Session MS` waits to ack message if not received within a time limit, it retry again.
* If retry count exceeds publish message to `offline message`.
* If Ack received after timeout, `offline message` is published for Ack.
* If user is not connected.
* Publish message to `offline message`.
* `Push notification ms` listen for offline message and send push notification.
* `Persistence ms` listen for offline message.
* If message is not ack, Store in the database until user comes online.
* If message is ack, remove the stored messages.

#### Group Chat

##### Create New Group

**Client Side**

* Choose the participants and group name.
* Generate an unique group id.
* Save group info in local store.
* Send http request to create group. (if connected).
* Publish message using rest endpoint from background service.

**Server Side**

* **Sync/Online** flow will remain the same as old one.
* **Async/Offline** flow

  * `Gateway` publish `onMessage` event to message broker to meta info like from.
  * `Message Router` listen for `onMessage` and forward to `Group message Router`.
  * `Group Message Router` parse message and call `Group MS` to create group.
  * `Group MS` verify participants and create new group.
  * Publish new `send message` for participants.
  * Further flow will be same as 1 to 1 chat.

##### Update Group

**Client Side**

* Choose participants to add or remove.
* Save group info in local store.
* Send http request to add/remove from group. (if connected).
* Publish message using rest endpoint from background service.

**Server Side**

* **Sync/Online** flow will remain the same as old one.
* **Async/Offline** flow

  * It will be handled similar to create group flow, with exception that instead `create group` call by message handle it will be `update group`.

##### Send Message (Group Message).

**Client Side**

 This will be same as 1 to 1 chat flow.

**Server Side**

All the steps are similar to 1 to 1 chat flow, other than the following mentioned below

* `Message Router` publish message to `group message` for `Group Message Router`.
* `Group Message Router` parse the message
* Fetch the users for the group
* Update the message META with target users.
* Publish message to `send message` event to message broker.

## **Let's see how new version of the system solves the existing problems.**

* #### No proper acknowledgement system

  Now the client needs to send the ack message against the message id to mark the message as delivered else the system will retry to send the message. If failed it will be treated as offline message and will be stored in database and push via firebase.
* #### Only web socket gateway for events, making it difficult for background event syncing.

  With new `Rest Gateway` and `Persistence store rest endpoint` background event syncing is possible.
* #### Router publishing directly into gateway topic, making message delivery logic to be duplicated among services.

  Responsibility to route message to gateway now has been move to `Session MS` which will act as the common point of contact to send the message to client. 
* #### Group operations are synchronous making it not useable offline.

  Group operation as now async using the `Rest Gateway` and `Group message router` new update.

I have start working on incorporating the required changes in the system, It will be available soon.

If you have any have suggestions or questions feel free to reach out to me on twitter [@ramank775](https://twitter.com/@ramank775).