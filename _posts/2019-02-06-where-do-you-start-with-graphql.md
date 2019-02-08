---
title: "Where do you start with GraphQL? I asked four engineers"
categories:
  - Mobile
---

There has been a lot of discussion on GraphQL. In time, it may rank up there alongside REST as a defining protocol for client-server computing. It is at least trending that way right now. REST has got longevity going for it — years of top engineers thinking about the best ways to structure a REST-based API and thoughts on how to handle it. GraphQL hasn’t got the longevity. It does, however, have senior professional developers who constantly think about APIs and development in order to answer those questions on structure and best practices.

I am not one of those developers.

I do, however, know quite a few developers, so I decided to ask four of them to philosophize on the state of GraphQL, what was hard in learning GraphQL and how you can make it easier on yourself, and best practices for developing a GraphQL API.

## Two non-GraphQL engineers thoughts

Chandra Bommas is a Principal Engineer with the AWS Mobile SDK for iOS and Android teams, predominantly from an API development background. He is not a “GraphQL” person, but rather an API person. To him, GraphQL solves the two main problems associated with mobile API.

Chandra has an overarching philosophy for API design. Take the work you need to do within the API; make it easy on the consumer of the data and do the work wherever you produce the data. As a technologist, this appeals to him.

> Make it easy on the **CONSUMER** of the data, and do the work with the **PRODUCER** of the data. - _Chandra Bommas_

So, what are some of the problems when learning GraphQL? The language models information as a set of interconnected bubbles; when done, you see a graph. That graph does not care how the data is stored. “When I grab data from the graph, I can express what I need using constraints on the graph.” There was a chasm of knowledge to be crossed before Chandra could really understand GraphQL. The problem is conceptual. Don’t look at GraphQL as an alternate to an old API — POST and GET doesn’t map to mutations and queries. You need to throw out the old model and embrace the information graph.

You should also not think about GraphQL as a database. While the concepts are similar on the surface, they soon break down. As a user of GraphQL (for example, inside a mobile app), you need to break free of the self-imposed shackles where you constrain your models based on how the data is stored, and work only with the information graph.

> Leverage tools such as Hasura, AWS AppSync, or Prisma to get started fast. Then you can concentrate on your app. — _Nader Dabit_

As the host of React Native Radio and a Developer Advocate for AWS Amplify, [Nader Dabit](https://twitter.com/dabit3) is well known in the React Native community as an educator, but his engineering credentials are solid as well. He is primarily a front end developer. For him, doing the backend database pieces and hooking the resolvers to databases was the most confusing part. “Of course, AWS AppSync now makes this effortless”, he remarked. He didn’t worry about the information graph as he found early on that it is self-documenting. He found leveraging tools like Hasura, AWS AppSync, or Prisma allows you to quickly get a functional GraphQL service off the ground and allows you to start playing with data in the front end app.

One thing to concentrate on? When developing your client, pay particular attention to nullable types in the GraphQL schema. The main issue is to ensure that the server gets the right information to fulfill the request. Nullable types are the default in GraphQL, and that makes a good API versioning story, but it can easily lead to obscure problems if you are not careful.

## Two GraphQL engineers thoughts

> Look at an open API (like GitHub) with GraphIQL to get familiar with the GraphQL Language — _Michael Paris_

[Michael Paris](https://twitter.com/mikeparisstuff) started Scaphold prior to joining AWS on the AWS AppSync team and has been working with GraphQL services since Facebook open-sourced the technology in late 2015. His thoughts?

One of the tools that made GraphQL popular is GraphiQL — a web-based GraphQL API explorer. The impact it has on being able to understand an API is amazing. Go look at an open API (like GitHub) with GraphiQL and get familiar with the language.

Well-defined operations are important. Once you get into more complex scenarios, you have to decide between more focused operations vs. a general query. While general queries (such as `searchAllRestaurants`) with an expansive input type for arguments is tempting, more focused operations (such as `searchRestaurantsByName`) are easier to optimize. There is no namspacing in GraphQL, so use long unambiguous names for your operations and types.

Really think how you will handle authorization. For larger organizations, push authorization to the service, but for smaller organizations, it’s ok for authorization to be placed in the GraphQL schema.

Use the errors response properly and deal with a partial response. REST tends to either succeed or fail and can be handled using the HTTP response codes. With GraphQL, you may be authorized to access one part of the graph and not another, so errors and a partial result are quite possible.

One thing Michael wished he had known or delved into earlier was NoSQL. If you are going to store data in NoSQL, then you have to think about how that impacts your service. It’s a shift in mindset on how to model data and how to query the NoSQL data within GraphQL resolvers.

> Figure out not only the data model but also the access patterns. — _Michael Paris_

Finally, before you start coding, figure out not only the data model but also the access patterns. This will allow you to design an efficient schema for your service.

As the principal engineer on AWS AppSync, [Rohan Deshpande](https://twitter.com/appwiz) has seen and solved his share of developer issues. He likes looking at GraphiQL as a learning tool. He started his learning using [howtographql.com](https://howtographql.com) — a site run by Prisma (so it’s a little biased) but still with excellent information. Going through the full tutorial allowed him to ramp up quickly. His major problem is learning was getting into native development and the Apollo client too quickly. The Apollo client has a lot of extra considerations that just aren’t important when you start learning about the technology. He got much more traction when he switched over to a web-based app for learning.

> GraphQL is about only asking for what you need. It doesn’t matter if it’s a query, mutation, or subscription — never ask for more than you need. — _Rohan Deshpande_

Rohan sees a lot of customer issues that are generated because the developer didn’t consider the user experience. For example, developers don’t take advantage of batching. Rather than doing 5000 mutations, restructure your schema to provide the ability to submit batches of 25 items. Your user gets a more responsive app and you reduce the number of calls, which reduces the drain on the battery and the network requirements.

> Don’t look at the API just from the perspective of an API call. Include the user experience in your API design — _Rohan Deshpande_

GraphQL is primarily a client-server (as opposed to server-server) API surface, so you should always include the impact on the user experience. It may be responsiveness, network utilization, or battery drain. There is almost always a user experience effect when you design such an API.

## Wrap up

There are definitely some best practices:

* Get to know GraphQL by looking at real services using GraphiQL.
* Follow the experts and read the tutorials (but only the latest tutorials).
* Spend time understanding the information graph
* Simplify the backend setup using services and concentrate your learning on the front end.
* Web/JavaScript apps have more tooling and simpler APIs than native apps, so start there first.
* Ask questions — you will be surprised how many people are willing to help.

There are also some well known worst practices:

* Skip the older tutorials as they are likely to be out of date.
* Don’t assume your knowledge of REST or other APIs is translatable to GraphQL.
* Client libraries may help in the long term, but they can obscure complex interactions that you need to understand.

Happy learning!
