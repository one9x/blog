---
layout: post
title: "Vartalap: Chat Server Architecture V2.1"
date: 2021-06-26T07:55:14.558Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This is 4th blog post in series of blogs posts, where I will be sharing details about Vartalap. How it started, decision I make while development, technical details, problem with current system and how I am planning to solve it. In this blog I'll be discussing the new iteration of improvements in the backend architecture v2. If you haven’t already read the previous one where I have shared the v2 of the architecture, I highly recommend reading it before proceeding further [Vartalap: Chat Server Architecture V2](https://blog.one9x.org/vartalap/2021/05/22/vartalap-chat-server-architecture-v2.html).

## Problem Statement

In the v2 architecture, most of the problems which are there in v1 are almost resolved like

* No proper acknowledgement system
* Background sync
* Offline group operations
* Common service handling message routing responsibility

However with v2 architecture where `session-ms` has got the additional responsibility including taking care of the acknowledgment handling. Where it suppose to save the message until the acknowledgment is received or timeout occur and then pass it to for persistence storage. If an acknowledgment comes after the timeout then it `persistence storage ms` will be handling it and removes the stored message.
Similarly when a when user came online, `persistence storage ms` will check if there is any stored message and then send back it to `session-ms` to forward it the user.
In the both cases there is overlapping in responsibility like acknowledgment handling, user offline message handling and handling user connect state which leads to duplication of code just like in v1 where routing responsibility is duplicated across multiple service and it adds operational and well as development overhead.

Other than overlapping responsibility, the logic to push the message for `persistence storage ms` which wait for timeout for acknowledgment is unnecessary add up the complexity and external resource to deal with it. 
For e.g. 

* Using internal function for timeout like `setTimeout` in JavaScript the message will be lost as service restarts.
* Using external service like `Redis` timeout for key using subscription, firstly it add an external resource to managed and also if the service is down for a particular period of time and the keys which are expired within that period will be lost.

## Architecture v2.1

New Architecture update:

![Architecture v2.1](/images/uploads/architecture-v2.1.png "Architecture v2.1")

In this iteration, both `session ms` and `persistence storage ms` are merged into a single new service `message delivery ms` which will be handling responsibility of message delivery in real time when user is connected, syncing messages when user is offline. Having a common service which is responsible to handling messages delivery make the system simpler and overcome existing challenges.

#### Responsibility of Message delivery service

* Maintaining User connection state.
* Message push in real time when user in connected.
* Store message when user is disconnected.
* Guarantee message delivery to receiver.
* Ability to sync message in background.

### Updated Request flow

***Request flow for client will remains the same.***

**Server Side**

* Gateway publish onMessage event to message broker to meta info like from.
* Message router listen for onMessage.
* Parse the message and extract destination information.
* Publish message to send message event to message broker.
* `Message delivery ms` listen to send message.
* Message will be saved in persistence storage.
* If user is connected.

  * It will send message to gateway using Gateway send endpoint.
  * Gateway tries to send message to user socket.

    * If failed, return failure response.
    * `Message delivery ms` retry again.
    * If retry count exceeds message mark as undelivered.
    * Publish message into `offline message topic`
  * If success, 

    * Wait for ack and delete the message from persistence storage.
    * If ack not received within a particular time, retry message
    * If retry count exceeds, publish message into `offline message topic`
* If user is not connected.

  * Mark message as undelivered
  * Publish message into `offline message topic`
  * `Push notification ms` listen for `offline message` and send push notification.
* When user came online, check if any message pending to be send and send those messages.

## Benefits

* No data loss.
* No overlapping responsibility. 
* Reduce the complexity as there is no need for listen to timeout for acknowledgment to store message for offline as now the logic is as simple as removing message as acknowledgment received.
* Less moving parts, as two services are merged into a single one with maintaining the segregation of responsibility.
* `Message delivery ms` name seems to be more suitable for the operation then `session-ms`.



## Tradeoff and possible solution

* High frequency database operation insertion and deletion of messages

  This can become bottleneck when the load is high, however it can be resolved by using multi level storage solution i.e. where first message will be stored in memory, then after certain time period it will be added to disk storage. Most of the real time operation can be handled when data is in memory or faster access storage leads to better performance and fewer operation on data which is stored on disk. Something similar to [Facebook Iris](https://engineering.fb.com/2014/10/09/production-engineering/building-mobile-first-infrastructure-for-messenger/)


If you have any have suggestions or questions feel free to reach out to me. My contact links are available on my bio page [Raman](https://blog.one9x.org/authors/raman.html).