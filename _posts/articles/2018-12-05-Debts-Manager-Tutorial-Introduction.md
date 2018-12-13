---
layout: post
title: "Debts Manager tutorial Part 1: Introduction"
excerpt: "Tutorial to build a complete Web API with Vert.x"
clippy: "Part 1 - Introduction"
modified: 2018-12-13T09:10:00+02:00
categories: articles
tags: [development, web, web api contract, vertx, openapi, openapi 3]
share: true
ads: true
---

Some months ago I decided to create a complete Vert.x application to show you capabilities of Vert.x for building Web APIs and, at the same time, I wanted to try some patterns I never used or applied. I'm going to create a _production ready_ application to finally manage the debts with my house mate with a fully powered Vert.x application!

Some notes before starting: I'm going to make this guide as complete as possible, but keep in mind that this is a side project and It could contain bugs and It could be incomplete. I will try to cover all interesting aspects about API design, implementation, testing and I will show you how I implemented Event Sourcing and CQRS. I don't plan to write a frontend for it (I don't want to hurt your eyes), but if you want to help me I'm glad to accept it!

The code is already available on [GitHub](https://github.com/slinkydeveloper/debts-manager) but It could change while I'm writing the guide.

## What Debts Manager should do

The purpose of Debts Manager is to manage the debts between two users of the service. The idea is similar to [Splitwise](https://www.splitwise.com/), but it will support only bills between two users. Every user should be registered to use the application. Then, if you want to receive bills from another user, you must **connect** to that user. When you are connected, you can bill him creating a transaction. For example:

* User A registers to the platform
* User B registers to the platform
* User B allows user A to bill himself. It does connecting to user A
* User A bills user B of 5 Euros for last grocery shopping

The final result is: user B now has a debt of 5 Euros with user A. Debts Manager will show to both users their status with various debts/credits

The connection between users are unidirectional, which means that if users want to bill each other they must create two diffent connections. There is no group concept, I wanted to keep things as simple as possible.

## Design

Before going further I want to show you a couple of things of the overall design of the application. These are required to undestand various aspects of the tutorial.

### Persistence & Event Sourcing

For persistence I choose PostgreSQL to store my data. The application stores into the database:

* The users instances (_user_)
* The connections between users (_user relationship_)
* The bills (_transaction_)

The DB access is provided by the blazing fast [reactive-pg-client library](https://github.com/reactiverse/reactive-pg-client)

The application stores the _transactions_ between users (events). You can use it as a log of various bills, but you also want to look at a summary of various credits/debits between connected users. To build it, we need to aggregate various transactions into one single structure that I call _status_. Every user has a status and is represented as a map with users as keys and total debts\credits as values. This map is built incrementally every time a user adds/modifies/removes a transaction and is stored in a Redis cache.

### Web API

The application exposes a Web REST API that you can interact with. It is documented with an OpenAPI 3 file and exposes most of CRUD endpoints for users, user connections and transactions (some are missing to keep things simple). It also has an endpoint to access status of users. The endpoints are protected with JWT tokens, so to use the application you must complete a login request and you get a token to use for the following requests. The Web API is implemented using [vertx-web](https://vertx.io/docs/vertx-web/java/), [vertx-web-api-contract](https://vertx.io/docs/vertx-web-api-contract/java/) and [vertx-web-api-service](https://vertx.io/docs/vertx-web-api-service/java/).

### Testing

Okay I admit it, I'm lazy :smile: I tested only the minimum features! I built these tests primarly to show you how I faced and solved common async test problems. I used Junit5 together with [vertx-junit5](https://vertx.io/docs/vertx-junit5/java/) and [testcontainers](https://www.testcontainers.org/) to spin up Redis and PostgreSQL.

## Tutorial parts

1. **Contract design**: Design the OpenAPI 3 contract
2. **Vert.x Web API Contract & Service**: Setup Vert.x project and bind Vert.x Event Bus services
3. **Persistence**: Design and implement persistence
4. **Event Sourcing**: Develop the read model and CQRS
5. **Testing**: Spin up test containers and write clean assertions
6. **BONUS: Deploy to OpenShift**
7. **BONUS: Refactor to microservices using Vert.x Event Bus**

Stay tuned for next chapter! And give me feedback about this tutorial!

