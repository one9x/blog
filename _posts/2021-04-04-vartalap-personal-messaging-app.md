---
layout: post
title: "Vartalap: Open Source Personal Messaging App"
date: 2021-04-04T09:48:45.860Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This blog post is part of series of blog post, where I will be sharing details about [Vartalap](https://vartalap.one9x.org) how it started, decision I make while developing it, technical details, problem with current system and how I am planning to solves it. In this blog I will be sharing introduction of Vartalap, motivation behind development and how it started.

## Introduction

[Vartalap](https://vartalap.one9x.org) is an open source personal messaging application just like Signal, WhatsApp, Telegram, etc. It is presently available on the google playstore [here](https://play.google.com/store/apps/details?id=com.one9x.vartalap) only in India. Also Vartalap is now part of [One9x](https://one9x.org)* *which is a non-profile organization, started with aim to develop the open source alternative of the commonly used application.*

**Note:** *One9x is started by myself along with some of my friends. At the time of writing this blog it's not a officially registered.*

## Motivation

The main motivation behind starting the Vartalap is never to develop an personal messaging application, but it always about learning something new skills and testing my skills on the real world use case. The complete idea of making an mobile applications comes along. Originally it started as a fun learning project while exploring [Kafka](https://kafka.apache.org) message broker and experimenting my skills of architecting a system which can scale upon real world benchmarks.

## How it started?

While playing around with Kafka and I decided to develop a chatting application with it. When starting the development of chat backend server, I took sometime to finalize the architecture and technology (I'll be sharing technical details in next blog post). I decided to go with microservice architecture other than the inherited benefits  of microservice there is another reason that was in my mind to reusability - having a independent services doing it works it will be easy to use them in any other future projects as of then there is no plan to launch it in public. The architecture is heavily inspired by the architecture discussed by [Whatsapp System Design: Chat Messaging Systems for Interviews by Gaurav Sen](https://www.youtube.com/watch?v=vvhC64hQZMk) and [Whatsapp System design or software architecture by Tech Dummies Narendra L](https://www.youtube.com/watch?v=L7LtmfFYjc4) with some modification. Backend is written in Nodejs which comes naturally to it, async nature of the system and no CPU bound operations link to source code [here](https://github.com/ramank775/chat-server).

It comes out to be pretty well with a Web app in Vuejs (Link to Vuejs app repo [here](https://github.com/ramank775/chat-ui)). After sharing it with friends and Manager at work, their feedback motivates me to proceed further with it. One common feedback I received is being a personal message application, a web application becomes the blocker in the adoptions. 

Which it true as it doesn't make sense to open a website to message someone, I thought of solution using PWA, which solves most of the smaller challenges like having launch icon, push notification, but one major issue is of data persistence although there is Indexed DB but data in indexed DB will got deleted as soon as someone decided to clear website data. So finally it is decided to develop an mobile application of it also another reason for not going with PWA is quite a personal choice, mainly to learn something new. 

Being backend developer and having close to zero knowledge of mobile application development, exploring the various frameworks majorly React-Native, Flutter. I just decided to go with flutter (Main reason is google top search results which comes up when I was looking for open source WhatsApp clone 😋😋).

## Where it is Today?

Audience Stats:
- Total Users : 73 approx.
- Active Users: 25 approx.

As of today [Vartalap](https://vartalap.one9x.org) is live on google playstore [here](https://play.google.com/store/apps/details?id=com.one9x.vartalap) only in India. The reason for restricting to India is internationalization, presently it can only handles mobile number without country code for India only. Which tbh not a big challenge will possibly solves in future or if someone for community raise a PR for it will be really helpful. Also the reason for not publishing it on the Apple store are I don't have personal Mac and the developer fee of Apple 99$/year (approx. 7K INR) and for some reason I decided not to make that investment.

Backend infrastructure looks like VM on AWS and Azure with Mongodb as database from Mongodb atlas and Kafka cluster as managed service provided by confluent cloud. Don't judge me for such a strange infra setup, the only reason behind it is the money. When these providers are actually providing such generous free tier, why not to use them. Anyway if you are wondering I have missed out Google, well for sign-in with phone number google firebase authentication with phone number is used 😁😁.

All the backend Nodejs microservices are running using pm2 process managers and communicating with each other via Kafka or when sync communication is needed for some microservices they are hosted on same VM so it all on localhost via Json based TCP server.

Supported Features:

* One to One chat (can work offline)
* Group chat (required internet for creation/adding members/removing members)
* Text based messages only (file sharing it currently not supported).

The reason for not supporting having file sharing functionality is due to my inexperience with flutter I am not able to build out the functionality on the  app to handle all file related operations.

## Future Plans

* Redesign backend server architecture.
* Making Application offline first.
* Refactor flutter application and adding new features.
* Better production infrastructure.
