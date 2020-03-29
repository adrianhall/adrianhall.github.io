---
title: "Building a GraphQL API by Example: Restaurant Reviews (part 3: The Resolvers)"
categories:
  - Cloud
tags:
  - GraphQL
---

I'm in the middle of building a GraphQL API for a restaurant review app that I'm building. Thus far, I've produced the schema and built the backend. However, the two don't know how to talk to one another. Specifically, how does the GraphQL insert a new location into DynamoDB, or do a geospatial search across ElasticSearch? This is the job of the resolvers - turn a GraphQL query into something that the back end data plane can understand.

There are several concepts to understand along the way, so I will attempt to introduce each one in isolation. First, however, I need to get the basic configuration added to my Serverless configuration. I'm using the [serverless-appsync-plugin](https://github.com/sid88in/serverless-appsync-plugin) to the Serverless Framework for this. Here are the basics:

```yaml
custom:
  # AWS AppSync GraphQL configuration
  appSync:
    name: ${self:custom.api}
    authenticationType: AWS_IAM
    logConfig:
      loggingRoleArn: { Fn::GetAtt: [ AppSyncLoggingServiceRole, Arn ]}
      level: ALL
    schema: ./schema.graphql
    dataSources:
      - type: AMAZON_DYNAMODB
        name: DynamoDB
        config:
          tableName: { Ref: DynamoDBTable }
          iamRoleStatements:
            - Effect: Allow
              Action:
                - "dynamodb:PutItem"
              Resource:
                - { Fn::GetAtt: [ DynamoDBTable, Arn ]}
                - { Fn::Join: [ "/", [{ Fn::GetAtt: [ DynamoDBTable, Arn ]}, "*" ]]}
      - type: AMAZON_ELASTICSEARCH
        name: ElasticSearch
        config:
          endpoint: { Fn::Join: [ "", [ "https://", { Fn::GetAtt: [ ElasticSearchDomain, DomainEndpoint ]} ]]}
          iamRoleStatements:
            - Effect: Allow
              Action:
                - "es:ESHttpGet"
              Resource:
                - { Fn::GetAtt: [ ElasticSearchDomain, DomainArn ]}
    mappingTemplates:
      - type: Mutation
        field: addLocation
        dataSource: DynamoDB
        request: Mutation-addLocation-request.vtl
        response: Mutation-addLocation-response.vtl
```

This has defined the two data sources I am going to use (DynamoDB and ElasticSearch Service) and my first resolver mapping template for the `addLocation()` mutation.

In addition to this, I need to add specific permissions to the `UnauthRole` and `AuthRole` to determine what operations can be performed. For instance, I added the following to the `AuthRole` to allow any authenticated user to perform any GraphQL operation:

```yaml
AuthRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: ${self:custom.api}-auth
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal:
            Federated: cognito-identity.amazonaws.com
          Action: sts:AssumeRoleWithWebIdentity
          Condition:
            ForAnyValue:StringLike:
              "cognito-identity.amazon.com:amr": "authenticated"
    Policies:
      - PolicyName: AllowAuthenticatedAppSyncAccess
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Action:
                - "appsync:GraphQL"
              Effect: Allow
              Resource:
                - { Fn::Join: [ "/", [ { Fn::GetAtt: [ GraphQLApi, Arn ]}, "*" ]]}
```

You can restrict the operations that can be performed by adding specific resources to each role.

## Adding a Location

Let's start with the most basic of the operations - adding a location. The information provided in the mutation is as follows:

```graphql
input LocationInput {
  name: String!
  gps: GPSInput
  address: AddressInput
  phone: String
  email: String
}
```

A resolver mapping consists of a request and response mapping template. The request mapping template will turn the query or mutation arguments (plus other information contained in the context, such as the identity of the user) into the form that the resolver actually needs. In the case of the DynamoDB resolver, it needs a JSON object. The request object for a DynamoDB `PutItem` operation is [documented well](https://docs.aws.amazon.com/amazondynamodb/latest/APIReference/API_PutItem.html). Here is my request mapping template:

```text
$util.qr($context.args.location.put("owner", $context.identity.username))
$util.qr($context.args.location.put("lastUpdated", $util.time.nowISO8601()))
{
    "version": "2017-02-28",
    "operation": "PutItem",
    "key": {
        "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
        "typeName": $util.dynamodb.toDynamoDBJson("LOCATION")
    },
    "attributeValues": $util.dynamodb.toMapValuesJson($context.args.location)
}
```

Note how I name my templates - this allows me to quickly locate them. Mapping templates for AWS AppSync are written in Velocity Template Language (or VTL).

The response template just turns the result back into an object so it can be returned as JSON:

```
$util.toJson($context.result)
```

I name this one `common-response.vtl` since it is generally used a lot in response templates.

Let's switch back to the request template for a little bit. There are some things I want to point out:

1. DynamoDB has a specific way of representing data values. A data value includes both a type and a value. For example, a string is written like this: `{ "S": "LOCATION" }` To handle this, there is a utility helper called `toDynamoDBJson()` that converts to this formation for you. You should always use this and the companion functions (like `toMapValuesJson()` for maps).
2. I'm using `autoId()` to generate an ID since I'm only adding a location. You will note that I don't have an `updateLocation`. It's relatively easy to handle but you have some more authorization to do (mostly ensuring that only the owner can update the record).
3. I added two fields to the map that is stored in DynamoDB. The first is the current time (in an easily parse-able form), and the second is the identity of the user. I'm using `username` here, but you could use other fields (for example, the ARN of the user). For more information, check out [the documentation](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-context-reference.html#aws-appsync-resolver-context-reference-identity).
4. The AWS AppSync console has lots of standard templates that you can copy as a baseline for your own adjustments. In addition, you can check out the request and response mappings provided as part of the Amplify CLI for various operations by deploying a test API with model transforms.

It's important to test your API as you go along. When testing this API, I do it in the following way:

1. Log onto the [AWS AppSync console](https://console.aws.amazon.com/appsync/home).
2. Select your API, then the **Queries** tab.
3. Create a template query, add a query variable, and then run the query.
4. Validate the output is as expected.
5. Go to the DynamoDB console and validate the data is correct in the table.
6. Go to the Kibana console and validate the data is correct in the ES domain.

For me, the template query looks like this:

```graphql
mutation AddLocation($input: LocationInput!) {
  addLocation(location: $input) {
    id
  }
}
```

And the query variables looks like this:

```json
{
  "input":{
    "name":"The Sphere",
    "gps":{
      "longitude":-122.3340779,
      "latitude":47.6147408
    },
    "address":{
      "street":"2101 7th Ave",
      "city":"Seattle",
      "state":"WA",
      "zipcode":"98101"
    }
  }
}
```

Hit run and you should get something akin to this:

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture1.png)

The ID will change in your situation since it is generated by the system. This result indicates that something worked. When I look at the record within DynamoDB, I get the following:

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture2.png)

Checking the data within ElasticSearch reveals a similar story - the data got everywhere it should have gotten.

You can do the same thing for adding a review. The request mapping template will change, but the concepts are the same.

## Marking a Location as Favorite

Let's consider the case of marking a location as a favorite. We give the mutation a location ID and then get the new location back. There are a couple of ways we can do this, but I want to perhaps have a list of favorites, so I'm going to have to store this as a separate data type. That means I have to do two operations - one to store the favorite and another to return the location.

This is great opportunity to use one of the new features of AWS AppSync: Pipeline Resolvers. With a pipeline resolver, you define a number of _functions_, each one of which does one operation. In this case, I will have two operations - one for storing the data and then a second one for retrieving the resulting location.

Building pipeline resolvers is a multi-step process. I do this in the console prior to transferring the resolvers to the `serverless.yml` file. Let's first consider what we are going to do.

* Obtain the location specified within the `locationId` field using a DynamoDB `GetItem` operation.
* If the location exists, then use `PutItem` to store a FAVORITE record within the DynamoDB table.

We need to do it this way round because we don't know that the location exists until we do the `GetItem` to retrieve it. Each operation will map to a specific function which is then converted into a pipeline. Whatever is returned by the response mapping template from one function will become `$context.prev.result` in the next step of the pipeline. The identity, source, arguments, and other parts of the context are passed through the pipeline intact.

To create this pipeline:

1. Open the AWS AppSync console and select the API.
2. Select **Schema**.
3. Find the _markFavorite_ mutation, then click **Attach**.
4. Click **Convert to pipeline resolver**.
5. Click **Save resolver**.

This will give you a request mapping and a response mapping for the overall resolver, plus an option to add functions. I want to store the `locationId` and the owner within a "stash" for the request. My request mapping looks like this:

```
$util.qr($context.stash.put("locationId", $context.args.locationId))
$util.qr($context.stash.put("owner", $context.identity.username))
{}
```

I could also put these inside the JSON object that the request mapping template is returning - this becomes the `$context.prev.result` for the first function in the list. It really depends if you want the data element to be available throughout the resolver or just to the first function. Anything you place in `$context.stash` is available from that point on through the life of the GraphQL request.

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture3.jpg)

The response mapping is the same as the `common-response.vtl` file I am already using and this is fine!

Let's continue by creating a function:

1. Click on **Functions** in the side bar.
2. Click **Create function**.
3. Select **DynamoDB** as the data source name and give the function a name (for example: `getLocation`)
4. Fill in the request and response mapping (see below)
5. Click **Create function**.

For the `getLocation` function, I'm using the following as the request mapping template:

```
{
    "operation": "GetItem",
    "key": {
        "id": $util.dynamodb.toDynamoDBJson($ctx.stash.get("locationId")),
        "typeName": $util.dynamodb.toDynamoDBJson("LOCATION")
    }
}
```

Don't forget to allow AWS AppSync to execute the `dynamodb:GetItem` action on the DynamoDB table in IAM. The response mapping is interesting:

```
## Raise a GraphQL field error in case of a datasource invocation error
#if($ctx.error)
    $util.error($ctx.error.message, $ctx.error.type)
#end
## Pass back the result from DynamoDB. **
$util.toJson($ctx.result)
```

Normally, we would just use `$util.toJson()` to return the results. However, we must now deal with errors, so this also returns the error, which will abort the path. 

You can now return to the resolver and add the function you just created. Don't forget to save the resolver at the end.

The resolver is currently equivalent to a `getLocation` operation. As such, we can actually execute it within the Queries window to see what is happening:

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture4.png)

In this case, I executed the _MarkFavorite_ query and got back the results of the first function. My next function will be called `storeFavorite`, and will take the result of the prior function, store the favorite marker, and then return the result of the prior function. This also uses the DynamoDB data source, with the following request mapping template:

```
#set($id = $ctx.stash.get("locationId")+"-"+$context.stash.get("owner"))
{
  "operation": "PutItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($id),
    "typeName": $util.dynamodb.toDynamoDBJson("FAVORITE")
  },
  "attributeValues": {
    "owner": $util.dynamodb.toDynamoDBJson($context.stash.get("owner")),
    "locationId": $util.dynamodb.toDynamoDBJson($context.stash.get("locationId"))
  }
}
```

Note that I am constructing the ID in this case. This allows the `PutItem` operation to update the record if needed so that I will only ever have one favorite for a given location/owner combination. The response mapping template becomes a little more interesting:

```
## Raise a GraphQL field error in case of a datasource invocation error
#if($ctx.error)
    $util.error($ctx.error.message, $ctx.error.type)
#end
## Pass back the result from DynamoDB. **
$util.toJson($context.prev.result)
```

The $context.prev.result is the result from the previous function (i.e. the location that is being favorited) in the pipeline. Again, we have to deal with error handling to abort early.

If you run the same `MarkFavorite` query now, you will get a side effect - a record will be inserted into the DynamoDB table with the appropriate `owner` and `locationId`.

I can now transfer this into the `custom:appSync` of my `serverless.yml` file:

```yaml
mappingTemplates:
  - type: Mutation
    field: markFavorite
    request: Mutation-markFavorite-request.vtl
    response: common-response.vtl
    kind: PIPELINE
    functions:
      - getLocation
      - storeFavorite
functionConfigurations:
  - dataSource: DynamoDB
    name: getLocation
    request: Function-getLocation-request.vtl
    response: Function-getLocation-response.vtl
  - dataSource: DynamoDB
    name: storeFavorite
    request: Function-storeFavorite-request.vtl
    response: Function-storeFavorite-response.vtl
```

> Note that you will need to use v1.0.9 or later of the serverless-appsync-plugin to support pipeline resolvers. 

At this point, all the mutations I envisioned within the schema are done, so it's time to move to the queries.

## The "me" query

The first query that I needed to consider was the "me" query. The idea of this query is to have the ability to return "my data" - from the locations I have entered to the list of my reviews. However, the basic query is simple:

```graphql
type User {
  id: ID!
  name: String
  locations(paging: PagingRequest): LocationPagingConnection
  reviews(paging: PagingRequest): ReviewPagingConnection
  favorites(paging: PagingRequest): LocationPagingConnection
}

type Query {
  me: User!
}
```

Let's leave the locations, reviews, and favorites for now, and look at the `id` and `name` fields. The user ID is already configured within the system - we use it in the `storeFavorite` query as `$context.identity.username`. However, the name must come from somewhere. If we were using the Amazon Cognito user pools, we would be able to use a null resolver and pull the name from the Amazon Cognito claims that are stored within `$context.identity`. However, we are using IAM for authentication.

Another mechanism is to use a record within DynamoDB (with a `typeName` of USER and an `id` equal to the username) to store additional metadata about the user. There is a `GetItem` operation on DyanmoDB, so I can just execute that with the appropriate record as it may return an error. If I use the `2018–05–29` template version for the request, then `$context.result` will be null if the record does not exist. Using an earlier version within the DynamoDB request will return an error instead, which is undesirable in this case. I can use this change in the response mapping to decide what to return - some default data or the data from the DynamoDB table.

Here is my request mapping template:

```
{
  "version": "2018-05-29",
  "operation": "GetItem",
  "key": {
    "id": $util.dynamodb.toDynamoDBJson($ctx.identity.username),
    "typeName": $util.dynamodb.toDynamoDBJson("USER")
  }
}
```

And here is my response mapping template:

```
#if($util.isNull($context.result))
  #set($result = {})
  $util.qr($result.put("id", $context.identity.username))
  #return($result)
#end
$util.toJson($context.result)
```

I am using velocity template language (VTL) and an if clause to decide what to return. I should really add another mutation to my schema to allow the client to update the name as well. The `#return()` statement allows me to "break out" of the response mapping template early.

> You can also use `#return()` to break out of request mapping templates early. This allows you to return unauthorized results without executing a DynamoDB query, for example. For more information, check out [the directives section of the AWS AppSync documentation](https://docs-aws.amazon.com/appsync/latest/devguide/resolver-util-reference.html#aws-appsync-directives).

## Performing subqueries

The `me` query I introduced above deals with the user meta-data. However, it does not deal with locations, reviews, or favorites. For example, let's say I want to return "my" locations:

```graphql
query Me {
  me {
    id
    locations() { id name }
  }
}
```

The "locations" field here is a sub-query. It relies on the context of the outer query to fulfill the request. This is available through the $context.source variable within the mapping templates.

I can fulfill this request in one of two ways:

* I can use a DynamoDB Query to return pages of information. Query is a good thing, but Scan is less efficient, so if you end up doing a Scan, you may want to use the ElasticSearch query capability instead.
* I can use an ElasticSearch operation to return pages of information. This option has several additional search characteristics as well as just paging.

DynamoDB has a one or two-value primary key. I'm using a two-value primary key, composed of the _typeName_ and _id_ columns. The _typeName_ is the partition (or HASH) key, and the _id_ is the sort (or RANGE) key.  In addition, I can further filter the results using a filter. Learning the nuances of DynamoDB queries and scans allows you to construct highly scaleable searches.

ElasticSearch can do these queries with ease because of the way that data is indexed within ElasticSearch domains. I can also test the queries easily using the Kibana console, which makes it very easy to use ElasticSearch for all queries.

For this specific search, both options are viable. If I were to introduce geospatial search characteristics (as an example), then I would definitely want to use the ElasticSearch option. In this case, I'm going to use a DynamoDB query with the following request mapping template:

```
{
  "version" : "2018-05-29",
  "operation" : "Query",
  "query" : {
    "expression": "typeName = :typeName",
    "expressionValues" : {
      ":typeName" : $util.dynamodb.toDynamoDBJson("LOCATION"),
    }
  },
  "filter": {
    "expression": "#owner = :owner",
    "expressionNames": {
      "#owner": "owner"
    },
    "expressionValues": {
      ":owner": $util.dynamodb.toDynamoDBJson($context.identity.username)
    }
  },
  "limit": $util.defaultIfNull(${ctx.args.limit}, 20),
  "nextToken": $util.toJson($util.defaultIfNullOrBlank($ctx.args.nextToken, null))
}
```

I'm using the "default" paginated list response mapping template. I can use a very similar query for the reviews as well. When placing this response mapping template into the `serverless.yml` file, I use a common filename (just like the single response version).

One of the things I do to reduce API development time is to look for commonality between resolvers (such as the reviews and locations fields). This allows me to spend the time creating the query for one of them and then rapidly duplicate the work for the other similar fields.

## The favorites query

The other query I have not covered within the `User` record is the `favorites` query. If you will remember, we are storing a favorite marker - basically a `locationId` and an owner - in the same table as everything else. However, the `favorites` query returns a paged `Location` response. The data from the `favorites` list in DynamoDB does not include this information, so we have to go and grab it as a secondary lookup. This is a good example of needing a pipeline resolver.

In this case, we will have two functions. The first (named `getFavorites`) will do a paged list of the favorites for a particular user, in much the same way as the `locations` and `reviews` resolvers that we looked at previously. However, we will then pass that into a second resolver (named `getLocationForList`) that will do a second query on the locations for a list of items. Here is the request mapping template for the `getFavorites` function:

```
{
  "version": "2018-05-29",
  "operation" : "Query",
  "query" : {
    "expression": "typeName = :typeName",
    "expressionValues" : {
      ":typeName" : $util.dynamodb.toDynamoDBJson("FAVORITE")
    }
  },
  "filter": {
    "expression": "#owner = :owner",
    "expressionNames": {
      "#owner": "owner"
    },
    "expressionValues": {
      ":owner": $util.dynamodb.toDynamoDBJson($context.identity.username)
    }
  },
  "limit": $util.defaultIfNull(${ctx.prev.result.limit}, 20),
  "nextToken": $util.toJson($util.defaultIfNullOrBlank($ctx.prev.result.nextToken, null))
}
```

However, the magic happens in the response mapping template:

```
#if($ctx.error)
  $util.error($ctx.error.message, $ctx.error.type)
#end

#set($idList = [])
#foreach($item in $context.result.items)
  $util.qr($idList.add($item.locationId))
#end
#set($result = {})
$util.qr($result.put("ids", $idList))
$util.qr($result.put("nextToken", $ctx.result.nextToken))
$util.toJson($result)
```

Assuming there is not an error, this loops through the results, pulls out the `locationId` from the item, and stores it in the `$idList`. This is then returned as a list of ids:

```json
{
  "ids": [ "a588408b-d02d-4cbc-819b-72c33c55d1c0" ],
  "nextToken": null
}
```

If you have logs turned on, you will be able to see this even though the eventual query will fail because it is not returning the right shape. To get the records of those IDs, I can use a DynamoDB `BatchGetItem` request, but I have to construct the keys properly. Unfortunately, I have to specify a table name within the request, which means I will normally have to construct a separate pair of resolver mapping templates for each environment (dev vs. prod). Here is my request mapping template:

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

And here is the response mapping template:

```
## Raise a GraphQL field error in case of a datasource invocation error
#if($ctx.error)
    $util.error($ctx.error.message, $ctx.error.type)
#end

#set($result = {})
$util.qr($result.put("nextToken",$ctx.prev.result.nextToken))
$util.qr($result.put("items", $ctx.result.data.devRestaurantReviews))
$util.toJson($result)
```

Note that I use the paging response from the first function in the response from the second function to ensure that paging works as expected. In addition, the results from the `BatchGetItem` operation are stored in `$ctx.result.data.tableName` — so I have to alter the response here for the table as well.

## The Reviews for a location query

Let’s move our attention to the `Location` type. There are three fields that need our attention here:

* favoriteCount
* averageRating
* reviews

Each field will need a special query. However, the one I am going to concentrate on here is the `reviews` query. What happens when I execute the following query:

```graphql
query Me {
  me {
    id
    favorites {
      items {
        id
        reviews { items { id } nextToken }
      }
      nextToken
    }
  }
}
```

I want to include the reviews for a specific location within the response. I’ve already covered how to do a sub-query, but I casually mentioned the `$context.source` field that is available within the resolver mapping templates. In this case, I’m going to need that.

When I do this query, here is what happens:

1. `/Query/me` is executed and returns results Q1.
2. `/Query/me/favorites` is executed with `$context.source` set to Q1, returning result Q2.
3. `/Location/reviews` is executed with `$context.source` set to Q2.items\[0\].
4. #3 is repeated, cycling through each item in the `items` list.

This can result in a long query time as many queries are being done recursively. You should ensure that your clients do not do this sort of query. Rather, return the IDs of the locations, then retrieve the reviews for just that location. This (I have realized) will require a `getLocation(locationId)` query which executes a `GetItem` within DynamoDB.

> It’s normal to tweak the schema as you go along. Just make sure your changes are additive, not replacing functionality (and hence removing previous functionality).

Here is the request mapping template for the `/Location/reviews` resolver:

```
{
  "version" : "2017-02-28",
  "operation" : "Query",
  "query" : {
    "expression": "#typeName = :typeName",
    "expressionNames": {
      "#typeName": "typeName"
    },
    "expressionValues" : {
      ":typeName" : $util.dynamodb.toDynamoDBJson("REVIEW")
    }
  },
  "filter": {
    "expression": "#location = :location",
      "expressionNames": {
        "#location": "locationId"
      },
      "expressionValues": {
        ":location": $util.dynamodb.toDynamoDBJson($context.source.id)
   }
  },
  "limit": $util.defaultIfNull(${ctx.args.limit}, 20),
  "nextToken": $util.toJson($util.defaultIfNullOrBlank($ctx.args.nextToken, null))
}
```

The parent (the `Location`) has an `id` field. However, the review stores this in the `locationId` field. So `$context.source.id` will be the id of the location that we want to grab reviews for.

## Field resolvers for aggregations

The other two fields within the `Location` type that need special attention warrant that attention because they are aggregations. The first — `favoriteCount` — requires us to count the number of favorite records in the dataset that have the required `locationId`, and the second — `averageRating` — requires that we search for review records and average the rating within them. Both of these can be done using the ElasticSearch API.

Let’s start with the easier of the two — counting records. The basic form for this (which you can test within the Kibana console) is to do a `GET _count` with some criteria. For instance:

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture5.png)

We can turn this into an ElasticSearch resolver with a pair of mapping templates. Here is the request mapping:

```
{
  "version":"2017-02-28",
  "operation":"GET",
  "path":"/_count",
  "params":{
    "body": {
      "query": {
        "bool": {
          "must": [
            { "match": { "typeName": "FAVORITE" } },
            { "match": { "locationId": "$context.source.id" } }
          ]
        }
      }
    }
  }
}
```

I cut and paste the query from Kibana into the `body` section of the request mapping template, then inserted the variables I wanted. The response mapping template is just one line:

```
$util.toJson($context.result.count)
```

Note that the expectation is that `favoriteCount` is a number, so our template is just returning the count as a number.  Most resolvers expect a JSON object, but that isn't a consistent rule.  It depends on what is expected at the schema level.   I can do a similar thing for the average aggregation. In this case, I had to do some hunting for the appropriate syntax and then validate through Kibana:

![]({{ site.baseurl }}/assets/images/2018/2018-12-06-picture6.png)

This is then formulated into mapping templates in the same way as the count example.

## Next steps

There are still a whole bunch of resolvers that I need to configure so that my schema actually works. For example, each review has a user record and a location record that needs to be returned. These are each another query with another resolver. However, we’ve covered the basic design patterns that you need to use. Once that is done, I can move onto creating client applications for the API.

I also need to deal with one more field, and it’s a complex one — the `findLocation` query. That will be the subject of my next blog.

Before I leave, let me give you the resources that I have handy at all times when I am developing resolver mapping templates:

* The [AWS AppSync Resolver Reference](https://docs.aws.amazon.com/appsync/latest/devguide/resolver-mapping-template-reference.html)
* The [DynamoDB Query / Filter Reference](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Query.html)
* The [ElasticSearch Query DSL Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)

I refer to these documents constantly, reaching out to Stack Overflow when needed since everyone has had the same issues as I have.

You can see the work so far in [my GitHub repository](https://github.com/adrianhall/restaurant-reviews/tree/p3).
