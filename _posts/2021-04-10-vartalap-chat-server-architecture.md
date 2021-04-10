---
layout: post
title: "Vartalap: Chat Server Architecture"
date: 2021-04-10T07:00:00.607Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This is second blog post is part of series of blog posts, where I will be sharing details about Vartalap how it started, decision I make while developing it, technical details, problem with current system and how I am planning to solves it. In this blog I will be sharing existing backend server architecture. If you haven't already read the first blog, I highly recommend reading that. Link to first blog spot [here (Vartalap: Open Source Personal Messaging App)](https://blog.one9x.org/vartalap/2021/04/04/vartalap-personal-messaging-app.html).

If you had read the previous blog post, you known that Vartalap is started as a learning project for Kafka and to test my skills to architect a system. Before starting architecting that system, Always the first step will be the note down the functional and non-functional requirement. Writing down the requirements gives the better picture what do we need to architect. One thing which is there in my mind to I am not going to build the complete system in one go, So I selected few of the minimal requirement which is should be there so make the application useable or the MVP product.

## Functional Requirement

* Login with mobile number.
* 1 to 1 and group rich text messaging (Supports emoticons).
* Contact list.
* Offline availability.
* Can receive message when user existed the app.
* Messages shouldn't be stored on the server forever.

## Non-Functional Requirement

* Real time message delivery
* Scalable 
* Highly available
* Fault tolerance
* Data integrate

# Architecture

After doing few iteration of working on the architecture, finally I settled with this one.

![Vartalap Server Architecture](images/uploads/architecture.png "Vartalap Server Architecture V1")

It's an microservice based architecture where system is divided into small groups of individual service responsible to handling a particular feature of the system. 
The advantage of having microservice architecture are

* No single point of failure improves the system fault tolerance.
* Can easily scale the specific part of the system which ever is needed.
* Can easily upgrade/replace the any part of the system without bringing whole system down.
* Deploying the system parts as it builds instead of waiting for complete system to go up.

In fact due the microservices it helps me motivated to build the system. Instead of waiting to bring the whole system up, I work on particular parts in my free time then deployed it and share it with friends and family members. Continuous feedback from them keeps me motivated but continue the development.

## Components

* Nginx as API gateway
* Kafka as Message brokers
* Mongodb as database
* Firebase for push notification
* Redis as cache store
* Nodejs for microservices

## Services

List of microservices and responsibility:

* Profile service

  * Responsibility

    * Login
    * AUTH
    * Contact Sync
* Web Socket Gateway

  * Responsibility

    * Maintaining Web Socket Connection
    * Forwarding socket event like `onConnect`, `onDisconnect`, `onMessage` to message broker (Kafka)
    * Sending message back to client
* Session MS

  * Responsibility

    * Maintain User connection state
* Message Router

  * Responsibility

    * Route 1-to-1 chat message to gateway
    * Route message to store/push notification
    * Retry failed message
    * Route group message to Group Message Router
* Push Notification

  * Responsibility

    * Deliver message to offline user
* Persistence Storage service: To store message until it got delivered

  * Responsibility

    * Store message new user is offline
    * Deliver message as user come online
* Group MS

  * Responsibility

    * Create Group
    * Add, Remove Members
    * Fetch groups
* Group Message Router

  * Responsibility
  * Route Group Message to respective destination

## Request flows

### Authentication

Client Side:

* User login/signup with mobile number
* Send OTP using firebase authentication service
* Verify OTP using firebase authentication service
* Get firebase authentication token
* Send firebase authentication token to profile service for login

Server Side:

* Verify firebase authentication token
* Check if it's a new user then signup
* Generate authentication token
* Publish event for new login
* Return authentication token
* Push Notification service listen to new login event
* Update the FCM token against the user

### Startup

* On App open
  Client Side: 

  * Web socket connection request.
  * Contact Sync request.

  Server Side (Web socket):

  * Nginx authenticates via issuing sub-request to profile ms for auth.
  * Nginx forward request to web socket gateway.
  * Fetch user info for request header.
  * Gateway accepts the upgrade connection requests.
  * Raise event for `onConnect` to message brokers.
  * Store web socket connection against users.
  * Session ms listen for `onConnect` event and store information of user and gateway mapping
  * Persistence ms listen for `onConnect` event and fetch all the pending messages
  * Publish `onMessage` event for the user.
  * Delete all the stored messages.

  Server Side (Sync request):

  * Nginx authenticates via issuing sub-request to profile ms for auth.
  * Nginx forward request to profile ms for contact sync.
  * Profile ms verify the registered users from list of contacts in payload.
  * Returns list of app users from contacts in payload.

### Send Message (1 to 1)

Client Side:

* Send message against a chat
* Store message against chat in local store
* Publish message to web socket.

Server Side.

* Gateway publish `onMessage` event to message broker to meta info like from.
* Message router listen for `onMessage`.
* Parse the message and extract destination information.
* Call session ms for gateway server against the user.
* If user is connected.
* Publish message to message broker against gateway server topic (generally the name of the server).
* Gateway server listen for it's topic and tries to send message to user socket.
* If failed, publish message for failed message which will be against consumed by message router and follow same steps.
* If user is not connected.
* Publish message to offline message.
* Push notification ms listen for offline message and send push notification.
* Persistence ms listen for offline message and store in the database until user comes online.

## Group Chat

### Create New Group

Client Side:

* Choose the participants and group name.
* Send http request to create group.

Server Side:

* Nginx authenticates via issuing sub-request to profile ms for auth.
* Nginx forward request to group ms.
* Group ms validate the participants
* Create a new group
* Publish new `groupMessage` for participants.
* Returns `groupId`.
* Group Message router, fetch the connected gateway server info from session ms.
* Publish message to gateway topic. (Further flow will be same as 1 to 1 chat).
* For offline message published to offline message.(Further flow will be same as 1 to 1 chat).

### Update Group

Client Side:

* Choose participants to add or remove.
* Send http request for add or remove.

Server Side:

* Nginx authenticates via issuing sub-request to profile ms for auth.
* Nginx forward request to group ms.
* Group ms validate the participants
* Update the group
* Publish new `groupMessage` for participants.
* `groupMessage` will be handled similar to Create New Group.

### Send Message (Group Message).

Client Side:

* Send message against a chat
* Store message against chat in local store
* Publish message to web socket.

Server Side.

* Gateway publish `onMessage` event to message broker to meta info like from.
* Message router listen for `onMessage`.
* Parse the message and extract destination information.
* Publish message to `groupMessage`.
* `groupMessage` will be handled similar to Create New Group.


## Limitation of Current Architecture
- No proper acknowledgement system for message delivery.
- Only web socket gateway for events, making it difficult for background event syncing.
- Router publishing directly into gateway topic, making message delivery logic to be duplicated among services.
- Group operations are synchronous making it not useable offline.
- Base message template for events.

To overcome the limitation of current system I am working on the second version of the system. Details of which I'll be sharing on the next blog post.