---
title: "Build a Universal Search API with GraphQL and AWS AppSync"
categories:
  - Cloud
tags:
  - GraphQL
---

Have you ever looked at a feature of a mobile app and wondered how they do something? Me too! I like to figure out how they built those features and build them into my own apps. Take, as an example, universal search. You can find this sort of search box at the top of the Facebook app. So, how do they implement it? I’ve not seen their codebase, but I imagine it’s something similar to the following proof of concept.

## The schema

I have a GraphQL API already built within [AWS AppSync] with the following schema:

{% highlight graphql linenos %}
interface Item {
  id:ID!
  title:String!
}

type Post implements Item {
  id:ID!
  title:String!
  comments:[Comment]
}

type Comment implements Item {
  id:ID!
  title:String!
  postId:ID!
}

type PaginatedPosts {
  nextToken:String
  items:[Post]
}

type PaginatedItems {
  nextToken:String
  items:[Item]
}

type Query {
  search(query:String!, limit:Int, nextToken:String):PaginatedItems
  allPosts(limit:Int, nextToken:String): PaginatedPosts
  getPost(postId:ID!):Post
}

type Mutation {
  addPost(title:String!): Post
  addComment(postId:ID!, title:String!): Comment
}

schema {
  query:Query
  mutation:Mutation
}
{% endhighlight %}

There are some new things in here. The first one is an _interface_ (see lines 1–4. An interface in GraphQL is just like an interface in other languages. It specifies that the types that implement the interface must have certain fields. The side effect is that derivitive types (`Post` and `Comment` in this case) can be referred to as a single entity. The second item is a _search function_ (see line 29). This returns a paged list of `Item` objects (see lines 23–26). The items in the list could be either `Post` or `Comment` because they both implement the `Item` interface.

> This isn’t the only way to implement universal search on the server side. You should also check out unions in [the GraphQL specification](https://graphql.org/learn/schema/#union-types).

The schema is connected to a single [Amazon DynamoDB] table that stores both the posts and the comments. The DynamoDB table has a partition key of `typename` (used to store the type of data) and a sort key of `id` (used to identify each individual record). I’m using an API key for authorization in this instance, but I could easily authenticate users using [OIDC](https://docs.aws.amazon.com/appsync/latest/devguide/security.html#openid-connect-authorization) or [Amazon Cognito user pools](https://aws.amazon.com/cognito).

## The resolvers

Most of the resolvers are fairly straight forward. The resolver for `addPost` and `addComment` store the `typename` as well as generating the ID. For example, here is the `addComment` resolver:

{% highlight json %}
{
    "version" : "2017-02-28",
    "operation" : "PutItem",
    "key" : {
     "typename": { "S": "Comment" },
        "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
    },
    "attributeValues" : $util.dynamodb.toMapValuesJson($ctx.args)
}
{% endhighlight %}

Putting everything in one table is not the most efficient method of linking the data storage to the GraphQL schema. Some operations need to use a `Scan` or secondary indices to make them perform well. In particular, the `comments` resolver looks like this:

{% highlight json %}
{
    "version" : "2017-02-28",
    "operation" : "Scan",
    "filter" : {
        "expression": "#typename = :typename AND #postId = :postId",
        "expressionNames": {
          "#typename": "typename",
          "#postId": "postId"
        },
        "expressionValues" : {
          ":typename" : $util.dynamodb.toDynamoDBJson("Comment"),
          ":postId" : $util.dynamodb.toDynamoDBJson($ctx.source.id)
        }
    }
}
{% endhighlight %}

In this case, I have to use a `Scan` operation to get the appropriate data. This is less performant than a `Query` operation. I cannot use a `Query` because `postId` is just a regular field. I’m not here to design an awesome database, so these details would need to be looked at in a production implementation.

> If you are contemplating this schema for your own project, look into using two tables — one for each of `Post` and `Comment` — then feed the data automatically into a single ElasticSearch instance. Use the ElasticSearch instance to fulfill search queries. This will provide better search flexibility while storing the data in two different tables (one for each data type).

Back to the implementation. The one operation I have not considered yet is the `search` operation. I want to feed a query string in and search for items where the string is a part of the title. The `title` field is a part of the interface, so it will be available on every single record. Here is my first pass at the resolver:

{% highlight json %}
{
    "version" : "2017-02-28",
    "operation" : "Scan",
    "filter" : {
        "expression": "contains(#title,:query)",
       "expressionNames": {
         "#title": "title"
        },
        "expressionValues" : {
            ":query" : $util.dynamodb.toDynamoDBJson($ctx.args.query)
        }
    },
    "limit": $util.defaultIfNull(${ctx.args.limit}, 20),
    "nextToken": $util.toJson($util.defaultIfNullOrBlank($ctx.args.nextToken, null))
}
{% endhighlight %}

In this case, a Scan is appropriate because we want to search all the items for a random string. In a scalable solution, I’d use an alternate data source (for example, an ElasticSearch service) instead of DynamoDB as this is an inefficient search.

**Spoiler alert**: The request resolver works and the right query is executed, but the response resolver does not work. You will get the following error if you have added appropriate data and run a search operation:

{% highlight json %}
{
  "errors": [
    {
      "errorType": "BadRequestException",
      "message": "Could not determine the exact type of Item. Missing __typename key on value.'"
    }
  ]
}
{% endhighlight %}

## Updating the response mapping

When a GraphQL service sends a record back, there is an accompanying type stored in a field called `__typename`. Since we have an interface, AWS AppSync cannot determine the concrete type, hence it produces an error. We have to use the response resolver to add the appropriate type. Fortunately, each record has a `typename` field within the database that gives the appropriate type. We can iterate through the items and add the appropriate type before returning the final object. This is my response resolver:

```
{
  "nextToken":$util.toJson($ctx.result.nextToken),
  "items": [
    #foreach($entry in $ctx.result.items)
    $!entry.put("__typename", $entry.get('typename'))
    $util.toJson($entry),
    #end
  ]
}
```

Normally, we would just convert the response to JSON. Here, we are constructing the JSON output by iterating through the items, adjusting them for our requirements as we move through the items.

The `__typename` field is available via the normal GraphQL client libraries and you can make decisions based on this — for example, you may want to change the icon or pick a different `ViewHolder` in an Android RecyclerView configuration.

## Wrap Up

This is just one way that you can implement the service side of a universal search box. Using interfaces and unions within a GraphQL schema is a great way to get this sort of functionality. AWS AppSync makes this sort of functionality easy to implement.

{% include links.md %}
