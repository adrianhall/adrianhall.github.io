---
title: "Early return from GraphQL Resolvers with AWS AppSync"
categories:
  - Cloud
tags:
  - GraphQL
  - "zCloud:AWS"
---

I am currently developing a "restaurant reviews" app, written in React Native and using a suite of services surrounding [AWS AppSync] for the data backend. Yesterday, I ran into a problem. This is how I solved that problem.

First, let's take a look at the problem. I have a query that is submitted like this:

```graphql
{
  me {
    id
    name
    locations { totalCount }
    reviews { totalCount }
    favorites { totalCount }
  }
}
```

The query is used to generate a header on a "My Information" page. Eventually, I'll add other things to this query, but this is representative of the problem. Most of the items within this query are easily achieved. The ID comes from the Amazon Cognito authentication; the name comes from an Amazon DynamoDB lookup, and the counts of locations and reviews comes from an ElasticSearch query.

The favorites resolver is a little different. It is actually a combination of two functions that are executed in sequence. The first function uses an ElasticSearch query to get the total count and a list of IDs that are favorites. The second function uses a DynamoDB `BatchGetItem` to load the data for those favorites.

The function in question (`getLocationForList`) has a request mapping template:

```
#set($keys = [])
#foreach($id in ${ctx.prev.result.ids})
    #set($key = {})
    $util.qr($key.put("typeName", $util.dynamodb.toString("LOCATION")))
    $util.qr($key.put("id", $util.dynamodb.toString($id)))
    $util.qr($keys.add($key))
#end

{
    "version": "2018-05-29",
    "operation" : "BatchGetItem",
    "tables" : {
        "devRestaurantReviews": {
            "keys": $util.toJson($keys),
        }
    }
}
```

Here, `$ctx.prev.result` is the result of the prior step of the pipeline resolver. Here is the associated response mapping template:

```
## Raise a GraphQL field error in case of a datasource invocation error
#if($ctx.error)
    $util.error($ctx.error.message, $ctx.error.type)
#end

#set($result = {})
$util.qr($result.put("totalCount", $ctx.prev.result.totalCount))
$util.qr($result.put("nextToken",$ctx.prev.result.nextToken))
$util.qr($result.put("items", $ctx.result.data.devRestaurantReviews))
$util.toJson($result)
```

Again, `$ctx.prev.result` is the results from the prior step in the pipeline resolver (the one that gets the paged response), and `$ctx.result` is the results of the current step in the pipeline resolver (the one that resolves the list of IDs into complete objects).

## So, what's the problem?

The problem comes when a user first enters the app. They don't have any favorites in the list. As a result, `$ctx.prev.result.ids` is zero length. When the resolver tries to do `BatchGetItem`, the following error will be generated:

```
Object {
  "data": null,
  "errorInfo": null,
  "errorType": "MappingTemplate",
  "locations": Array [
   Object {
     "column": 5,
     "line": 11,
     "sourceName": null,
   },
  ],
  "message": "RequestItem keys '$[tables][devRestaurantReviews]' can't be empty",
  "path": Array [
    "me",
    "favorites",
  ],
},
```

The error is pretty clear. The list of IDs we are feeding into the BatchGetItem is empty, and it isn't allowed to be empty.

## How do we fix it?

To fix this, I need to introduce you to a new VTL directive: [`#return`](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-util-reference.html#aws-appsync-directives). The `#return` directive will exit out from the resolver, returning whatever you decide. You can also use `#return` on its own (with no argument) to return null.

Using `#return` in a mapping template of a function will return that data from the function. If you use it within a request mapping template, the response mapping template is bypassed. Similarly, using `#return` in a mapping template of a resolver will return that data from the resolver, prematurely ending the resolver execution.

My fix, then, is to alter the request mapping template such that it uses #return when the number of IDs is zero. Since VTL is based within Java and I am using an array, I can use the Java Array methods to find out information about the array. In this case, I can use `.size()` to determine the size of the array:

```
#if($ctx.prev.result.ids.size() == 0) 
  #set($result = {})
  $util.qr($result.put("totalCount", $ctx.prev.result.totalCount))
  $util.qr($result.put("nextToken",$ctx.prev.result.nextToken))
  $util.qr($result.put("items", $ctx.prev.result.ids))
  #return($result)
#end

#set($keys = [])
#foreach($id in ${ctx.prev.result.ids})
    #set($key = {})
    $util.qr($key.put("typeName", $util.dynamodb.toString("LOCATION")))
    $util.qr($key.put("id", $util.dynamodb.toString($id)))
    $util.qr($keys.add($key))
#end

{
    "version": "2018-05-29",
    "operation" : "BatchGetItem",
    "tables" : {
        "devRestaurantReviews": {
            "keys": $util.toJson($keys),
        }
    }
}
```

Note that I did not have to adjust the schema to accommodate this change in behavior. The front end can and should be isolated from the concerns of the backend data plane. Also, unlike most "responses", I don't convert to JSON - I just return the object that I want to return as the result.

The Velocity Template Language is incredibly powerful. Combined with pipeline resolvers, I would be hard pressed to find a situation that cannot be modeled with AWS AppSync.

{% include links.md %}