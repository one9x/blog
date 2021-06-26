---
layout: post
title: "Vartalap: Chat Server Architecture V2.1"
date: 2021-06-26T07:55:14.558Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This is a 4th blog post in a series of blog posts, where I will be sharing details about Vartalap. How it started, decisions I made while development, technical details, the problem with the current system and how I am planning to solve it. In this blog, I'll be discussing the new iteration of improvements in the backend architecture v2. If you haven’t already read the previous one where I have shared the v2 of the architecture, I highly recommend reading it before proceeding further [Vartalap: Chat Server Architecture V2](https://blog.one9x.org/vartalap/2021/05/22/vartalap-chat-server-architecture-v2.html).

## Problem Statement

In the v2 architecture, most of the problems which are there in v1 are almost resolved like

* No proper acknowledgement system
* Background sync
* Offline group operations
* Common service handling message routing responsibility

However, with v2 architecture where `session-ms` has got the additional responsibility, including taking care of the acknowledgment handling. Where it supposes to save the message until the acknowledgment is received or timeout occur and then pass it to for persistence storage. If an acknowledgment comes after the timeout, then `persistence storage ms` will be handling it and removes the stored message.

Similarly, when a when a user came online, `persistence storage ms` will check if there is any stored message and then send back it to `session-ms` to forward it the user. 

In the both cases there is overlap in responsibility like acknowledgment handling, user offline message handling and handling user connect state which leads to duplication of code just like in v1 where routing responsibility is duplicated across multiple service. It adds operational and well as development overhead.

Other than overlapping responsibility, the logic to push the message for `persistence storage ms` which wait for a timeout for acknowledgment is unnecessary add up the complexity and external resource to deal with it. 
For e.g. 

* Using an internal function for timeout like `setTimeout` in JavaScript the message will be lost as service restarts.
* Using an external service like `Redis` timeout for key using subscription, firstly it adds an external resource to manage and if the service is down for a particular period of time and the keys which are expired within that period will be lost.

## Architecture v2.1

New Architecture update:

![Architecture v2.1](/images/uploads/architecture-v2.1.png "Architecture v2.1")

In this iteration, both `session ms` and `persistence storage ms` are merged into a single new service `message delivery ms` which will be handling the responsibility of message delivery in real time when the user is connected, syncing messages when user is offline. Having a common service which is responsible for handling message delivery make the system simpler and overcome existing challenges. 

#### Responsibility of Message delivery service

* Maintaining User connection state.
* Message push in real time when a user in connected.
* Store message when the user is disconnected.
* Guaranteed message delivery to the receiver.
* Ability to sync message in background.

### Updated Request flow

***Request flow for client will remains the same.***

**Server Side**

* Gateway publishes an onMessage event to message broker to the meta info like from.
* Message router listens for `onMessage`.
* Parse the message and extract destination information.
* Publish a message to send message event to message broker.
* `Message delivery ms` listen for `send-message`.
* The message will be saved in persistent storage.
* If the user is connected.

  * It will send messages to the gateway using Gateway to send endpoint.
  * Gateway tries to send message to user socket.

    * If failed, return failure response.
    * `Message delivery ms` retry again.
    * If the retry count exceeds message mark as undelivered.
    * Publish message into `offline message topic`
  * If success, 

    * Wait for acknowledgment and delete the message from persistence storage.
    * If acknowledgment not received within a particular time, retry message
    * If the retry count exceeds, publish message into `offline message topic`
* If user is disconnected.

  * Mark message  undelivered.
  * Publish message into `offline message topic`.
  * `Push notification ms` listen to `offline message` and send push notification.
* When user came online, check if any message pending to be sent and send those messages.

## Benefits

* No data loss.
* No overlapping responsibility. 
* Reduce the complexity as there is no need for listening to timeout for acknowledgment to store messages for offline as now the logic is as simple as removing message as acknowledgment received.
* Less moving parts, as two services are merged into a single one with maintaining the segregation of responsibility.
* `Message delivery ms` name seems to be more suitable for the operation, then `session-ms`.



## Tradeoff and possible solution

* High frequency database operation insertion and deletion of messages can become bottlenecks

  This can become bottlenecks when the load is high, however, it can be resolved by using a multi level storage solution, i.e. where first message will be stored in memory, then after certain time period it will be added to disk storage. Most of the real time operation can be handled when data are in memory or faster access storage leads to better performance and fewer operation on data which is stored on disk. Something similar to [Facebook Iris](https://engineering.fb.com/2014/10/09/production-engineering/building-mobile-first-infrastructure-for-messenger/)


If you have any have suggestions or questions, feel free to reach out to me. My contact links are available on my bio page [Raman](https://blog.one9x.org/authors/raman.html).