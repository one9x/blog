---
layout: post
title: "Vartalap: Personal Messaging App"
date: 2021-04-04T09:48:45.860Z
image: images/uploads/vartalap-header.png
author: Raman
categories: Vartalap
---
This blog post is part of series of blog post, where I will be sharing details about [Vartalap](https://vartalap.one9x.org) how it started, decision I make while developing it, technical details.

## Introduction

[Vartalap](https://vartalap.one9x.org) is an open source personal messaging application just like Signal, WhatsApp, Telegram, etc. It started as a fun learning project while exploring [Kafka](https://kafka.apache.org) message broker and experimenting my skills of architecting a system which can scale upon real world benchmarks.

## How it started?

While playing around with Kafka and I decided to develop a small chatting application with it, which comes out to be pretty well with a Web UI in Vuejs. Link to Vuejs app repo [here](https://github.com/ramank775/chat-ui). After sharing it with friends and Manager at work, their feedback motivates me to proceed further with it. One common feedback I received is being a personal message application, a web application becomes the blocker in the adoptions. 

Which it true as it doesn't make sense to open a website to message someone, I thought of solution using PWA, which solves most of the smaller challenges like having launch icon, push notification, but one major issue is of data persistence although there is Indexed DB but data in indexed DB will got deleted as soon as someone decided to clear website data. So finally it is decided to develop an mobile application of it but being backend developer and having close to zero knowledge of mobile application development, exploring the various frameworks majorly React-Native, Flutter. I just decided to go with flutter (Main reason is google top search results which comes up when I was looking for open source WhatsApp clone 😋😋).