---
title: "Top Ten things to consider when taking your GraphQL service into production"
categories:
    - Cloud
tags:
    - GraphQL
---

It's a somewhat well-known facet of development that we don't consider production problems until it is too late in the development cycle.  When we look at taking a Web API into production, we use API management solutions to provide protection, control, and visibility into our APIs so that we ensure we don't get woken up by a production outage.  The things we need to consider are well understood in APIs in general, but what about GraphQL?  

GraphQL is a developers dream when it comes to tooling.  A lot of that tooling is driven by the underlying features of the protocol, like introspection.  As you move to production, you want to give up the flexibility that comes from the tooling and lock down the service so that you understand what operations are running, who is running them, and identify bad actors early.  So, what are some of the things you can do?  

This is not an article about how to write schemas, optimize operations, or operate GraphQL at scale (for example, as a multi-instance service).  There are plenty of other articles, books, and videos about those topics.  Rather, this is a pragmatic list for you to consider as you move your GraphQL API from development into production.

Here is my top ten.

1. Write input type validation
1. Implement user authorization
1. Implement per-client application keys
1. Control backend concurrency
1. Rate limit operations, not requests
1. Turn off introspection
1. Optimize the backend
1. Block abusive operations
1. Implement cost analysis and reject early
1. Implement Advanced Persisted Queries

Let's take a look at each one.

## 1. Write input type validation

When you are writing any database driven application, you validate inputs to ensure that the data is in the expected form.  SQL injection attacks are, thankfully, becoming a thing of the past since most people use some form of relational mapper (ORM).  However, that doesn't mean people won't try.  Your data store should not be doing the validation for you.  Ensure that every single argument and every single input type is validated before you call your data store.

Since I typically write in C#, I do this by implementing a validator class which is then called by the resolver to ensure the data I receive matches my expectations.  GraphQL gives us primitives - strings, numbers, and booleans.  We need to match that to the real requirements - an ID, a positive number in a range, or the existence of one or more limits.

## 2. Implement user authorization

You do authenticate your users, right?  Do you take it any further?  Do all your users have the same access, all the way down to the field level?  Do any of your results get limited based on user authorization.  These things need to be implemented and adjusted as early as possible.  I prefer to have per-operation and per-field limits based on authorization before any resolver gets run.

## 3. Implement per-client application keys

Even if your API doesn't ostensibly have authentication, you still want to understand which clients have access to your API and what they are doing.  You can implement API keys, or per-client application keys.  These can be implemented as a part of your client build - a key gets generated when the client gets built for release and then it gets baked into the configuration of the client.

There are a lot of benefits to this simple task.  If a key gets stolen (see #8 below), you can reject it.  You can lock down the operations a client can run based on its key (see #10 below).  You can even provide different limits (see #5 below) depending on requirements.  

You should not be adjusting your service for this functionality.  Rather, you should deploy an API gateway (my favorite is Azure API Management) that does this functionality for you.  That way you can have all the benefits of per-client application keys without implementing a line of code.

## 4. Control backend concurrency

There are two words that you need to be familiar with here - connection pooling.  You need to understand how many concurrent requests your service can make to your database services.  Each database service should have a connection pool associated with it and you should do all the requests to that service through the connection pool.  No connections available in the connection pool?  Your service is overloaded and wouldn't respond in a timely manner anyhow.  Your service should be returning a `503 Service unavailable` response to indicate that the service is too busy.

## 5. Rate limit operations, not requests

On the backend side of the API, we use concurrency controls.  On the client side, we use rate limiting, where the API service returns `429 Too many requests` when the client application is sending too many requests.  If you have an API gateway in front of your GraphQL service, you should do rate limiting in the API gateway.  

However, one thing to note here is that you can stuff multiple GraphQL operations in one API requests.  You should rate limit the operations, not the API requests.  This will ensure that the desired effect - a reduction in work when client applications send too much - is actually what is achieved.  When you rate limit the number of API requests, you miss an important attack vector for bad actors.

## 6. Turn off introspection

Talking of bad actors, have you ever thought about how much data you are giving to would-be attackers by leaving introspection turned on?  It turns out, introspection is as much a treasure trove of information to attackers as it is to your developers.  Production apps don't need introspection, so why would you need it left on?

If you want to provide introspection to only your developers, you can block introspection at the API gateway level.  Simply create an API key for your developers to use, and turn off introspection for everyone else.

## 7. Optimize the backend

One of the benefits of GraphQL is also one of its weaknesses.  In providing the "resolver" model, you introduce the N+1 problem.  The N+1 problem occurs because GraphQL executes a separate resolver for each field.  These resolvers can potentially make an additional round trip to the database.  Instead of using SQL JOINs, we use multiple requests.  This means that our database calls are not optimized and the number of requests to the backend database balloons, and with it the execution phase.

The general mechanism to solve the N+1 problem is to use [DataLoader](https://github.com/graphql/dataloader), which is a JavaScript specific library to batch requests going to the same data store.  Other platforms (for example, [Chilli Cream Hot Chocolate](https://chillicream.com/docs/hotchocolate/v12/fetching-data/dataloader)) have a similar functionality.

Another thing you may want to do is to implement caching (for example, through [Redis Cache](https://redis.com/solutions/use-cases/caching/)) to ensure that database results are cached at the resolver level.  You don't want to implement caching at the GraphQL response level, but caching the response to database requests is not only ok, but beneficial to the responsiveness to your service.

## 8. Block abusive operations

There are two types of abuse we need to deal with here.  Firstly, we want to deal with abusive requests from valid clients.  For example, let's say you have a query that allows the client to specify a count of the number of records to return to allow for paging.  What happens if the client specifies that you should return 10,000 records?  How about a million records?  What about if a million records each had an embedded object which returns another 10,000 records?  You should be implementing two controls here - first, an input type validation with an upper limit on the value of count, and secondly a "query depth" limit so that you don't get infinite recursion.  Most GraphQL services provide a query depth limit already, but you may want to further restrict this number.

The second type of abuse deals with bad actors.  A bad actor can potentially explicitly construct operations with the intent to overload your service.  You can solve this in multiple ways, and we are discussing a number of them here.  However, your main protection is to log every single operation and look for outliers.  Clients will inevitably generate the same operations over and over, so operations you have not seen before are of interest - they are likely to be a security concern.  You can either monitor or proactively block at the gateway.

## 9. Implement cost analysis and reject early

One of the ways of detecting abusive operations is to [calculate the cost](https://www.npmjs.com/package/graphql-cost-analysis) of operations prior to their execution.  To do this, you assign a weight to each resolver (which can be a static cost, complexity range, or multiplier for lists).  The execution engine will calculate the expected total cost of the operation and, if over a specified limit, reject the operation. You could also implement middleware that dynamically measures the cost of each resolver and works out the appropriate cost metric from historical averages.

Whatever the choice, you can then decide whether blocking the specific request is enough or if you need to block the IP address, client key, or user for repeat offenders.  As with most production protections, the more you can do automatically, the better you will sleep at night.

## 10. Implement Advanced Persisted Queries (APQ) 

Finally, let's talk persisted queries.  There are two types - one (popularized [by Apollo](https://www.apollographql.com/docs/apollo-server/performance/apq/)) is automatic persisted queries.  Automatic persisted queries are a mechanism to improve network performance for large query strings.  In a blank system, the client sends the hash of the query string, and the service sends an error.  The client then follows up with the query string and hash, which is then executed and returned.  After that, the client can just send the hash of the query string and the service now knows about it and can expand the query string from the hash.  This also allows you to integrate with a CDN to provide request level caching.

Advanced persisted queries skip the first step.  The allowed queries are programmed into the service; the client sends the ID of the query and the service expands it and executes it.  This is great because it allows the administrator of the service to lock down the queries that can be executed by clients.  You can even make a different query set available based on the client application ID to further restrict things.  You will no longer have to worry about bad actors sending invalid complex queries into the GraphQL API because they aren't allowed anyhow. Of course, this works best if done at the API gateway level so that complex queries never make it to your service.

So, there you have it - a generalized list of ten things you can do to secure, control, and monitor your GraphQL API in production.  Don't wait for bad actors and out of control client applications to give you a sleepless night dealing with the clean-up from an outage.  Be proactive by protecting your APIs.

