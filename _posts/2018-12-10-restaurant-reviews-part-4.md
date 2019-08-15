---
title: "Building a GraphQL API by Example: Restaurant Reviews (part 4: Geospatial Search)"
categories:
  - Cloud
tags:
  - GraphQL
---

I am building a GraphQL API for a restaurant review app, and thus far, I’ve:

* [Designed the schema]({% post_url 2018-11-15-restaurant-reviews %}).
* [Developed a suitable data layer backend]({% post_url 2018-11-30-restaurant-reviews-part-2 %}).
* [Developed “most” of the resolvers for the queries and mutations]({% post_url 2018-12-06-restaurant-reviews-part-3 %}).

When developing the resolvers in the last blog post, I left out what is probably the most complex and useful of the queries — the _findLocation_ query. This query is central to my app and involves geospatial searching. I included an ElasticSearch instance in my data layer backend purely for this capability.

## Implementing geospatial data storage

Before we get started, we need to configure the ElasticSearch Service search domain with geospatial support. The appropriate data type is a [_geo\_point_](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/geo-point.html) which represents a geospatial point with a longitude and latitude. There are a few ways to represent a geo-point within ElasticSearch indices, but the one we are going to use is an object:

```
"gps": {
  "lat": <some-float>,
  "lon": <some-float>
}
```

The fields **MUST** be called `lat` and `lon`. Nothing else will do. Fortunately, my schema already uses that form, so I can use those names. However, if you used (for example) `latitude` and `longitude` then you would have to rename those fields within the mapping template.

First, create the index using the Kibana Dev Tools:

```
# Create the index
PUT devrestaurantreviews
{
    "settings" : {
        "index" : {
            "number_of_shards" : 3,
            "number_of_replicas" : 1
        }
    }
}
```

Then adjust the mapping so that the `gps` field is a `geo_point`:

```
# Update to make the gps coords a geo_point for geospatial search
PUT devrestaurantreviews/_mapping/doc
{
  "properties": {
    "gps": {
      "type": "geo_point"
    }
  }
}
```

> You can also combine these two sections into a single call.  Refer to [the ElasticSearch documentation](https://www.elastic.co/guide/en/elasticsearch/reference/6.5/indices-create-index.html) for more information.

The `gps` field is the field within the `Location` type. The name of the field must uniquely represent a geo-point as all fields with this name (including those within other types) will be configured to be a geo_point. This also **MUST** be done prior to any data being stored in the index. As a result, you will want to wipe out your DynamoDB table contents and start over.

> You can use [`serverless-plugin-scripts`](https://www.npmjs.com/package/serverless-plugin-scripts) and execute a Lambda function on successful deployment that executes these commands. Check out [the GitHub repository](https://github.com/adrianhall/restaurant-reviews) for an example.

If you have already created the ES index, you can use Kibana Dev Tools to delete it:

```
DELETE devrestaurantreviews
```

Then re-create the index as above and store the mapping. Now, go over to the AWS AppSync console, add a location with GPS coordinates, then wait for the record to appear in the ES index. You can search for “all records” in Kibana Dev Tools like this:

```
GET _search
{
  "query": {
    "match_all": {}
  }
}
```

## Doing a geospatial search

Let’s continue in the Kibana Dev Tools and look at what a geospatial search looks like:

```
GET _search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "typeName": "LOCATION" } }
      ],
      "filter": {
        "geo_distance": {
          "distance": "10km",
          "gps": {
            "lat": -47.615,
            "lon": -122.330
          }
        }
      }
    }
  }
}
```

If I have any entries within 10km of the identified search location, then there will be “hits” that match.

## The findLocation request mapping template

Let’s now turn our attention to the _findLocation_ resolver. It actually has two fields — _byGPS_ and _byAddress_. Theoretically, these can be used in concert, so we need to know what that means. _byGPS_ is easy — it takes a set of GPS coordinates and a distance (measured in km), and constructs the filter. The _byAddress_ will try to exactly match the address fields that are entered. If you enter both, then the query will be ANDed together. Thus, we could end up with a search like this:

```
GET _search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "typeName": "LOCATION" } },
        { "match": { "address.zipcode": "98101" } }
      ],
      "filter": {
        "geo_distance": {
          "distance": "10km",
          "gps": {
            "lat": 47.615,
            "lon": -122.330
          }
        }
      }
    }
  }
}
```

Here is what I came up with for the request mapping template:

```

#set($query = {
  "bool": {
    "must": [
      { "match": { "typeName": "LOCATION" } }
    ]
  }
})

#if (! $util.isNull($context.args.byGPS ))
  $util.qr($query.bool.put("filter", {
  	"geo_distance": {
    	"distance": "${context.args.byGPS.radius}km",
        "gps": $context.args.byGPS.gps
    }
  }))
#end

#if (! $util.isNull($context.args.byAddress ))
  #if (! $util.isNull($context.args.byAddress.street ))
    $util.qr($query.bool.must.add({ "match": { "address.street": "$context.args.byAddress.street" } }))
  #end
  #if (! $util.isNull($context.args.byAddress.city ))
    $util.qr($query.bool.must.add({ "match": { "address.city": "$context.args.byAddress.city" } }))
  #end
  #if (! $util.isNull($context.args.byAddress.state ))
    $util.qr($query.bool.must.add({ "match": { "address.state": "$context.args.byAddress.state" } }))
  #end
  #if (! $util.isNull($context.args.byAddress.zipcode ))
    $util.qr($query.bool.must.add({ "match": { "address.zipcode": "$context.args.byAddress.zipcode" } }))
  #end
#end

{
    "version":"2017-02-28",
    "operation":"GET",
    "path":"/_search",
    "params":{
        "body": {
            "query": $util.toJson($query)
        }
    }
}
```

I construct the query object inside of a map in the VTL. The initial version is provided by the `#set` statement at lines 1–7. Note that this must look like JSON — i.e. the fields must be quoted. I continue by setting up the filter, but only if the _byGPS_ field is provided (which is signified by the value being not null). I do a similar process with the _byAddress_ fields. Since the sub-fields of the `AddressInput` type are also optional, I need to do null checks there.

I did not do a “paging request” for this search capability, but should have done so. If you add `from` and `size` fields to the body of the params then you will be able to request arbitrary slices of the result set. You can use this to implement the paging request. Again, check [the GitHub repository](https://github.com/adrianhall/restaurant-reviews/) for the details.

## The findLocation response mapping template

There is a shape that the response mapping template is expected to produce, and it does not match what comes back from the resolver. As a result, we have to construct it. Here is an example on what comes back from the service:

```json
{
  "took": 12,
  "timed_out": false,
  "_shards": {
    "total": 3,
    "successful": 3,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 1,
    "max_score": 0.5753642,
    "hits": [
      {
        "_index": "devrestaurantreviews",
        "_type": "doc",
        "_id": "613df4d0-1ea2-44f2-895e-698a41d1ef4a",
        "_score": 0.5753642,
        "_source": {
          "owner": "AIDAI6XOAOVZFVVPRZHBM",
          "lastUpdated": "2018-12-03T21:31:57.878Z",
          "address": {
            "zipcode": "98101",
            "city": "Seattle",
            "street": "2101 7th Ave",
            "state": "WA"
          },
          "name": "The Sphere",
          "typeName": "LOCATION",
          "gps": {
            "lon": -122.3340779,
            "lat": 47.6147408
          },
          "id": "613df4d0-1ea2-44f2-895e-698a41d1ef4a"
        }
      }
    ]
  }
}
```

I can see this format by executing a query in Kibana Dev Tools. This becomes `$context.result` in the response mapping template. I can adjust to produce the right form as follows:

```
#set($items = [])
#foreach($entry in $context.result.hits.hits)
  $util.qr($items.add($entry.get("_source")))
#end
{
	"items": $util.toJson($items)
}
```

Still to do here: if I want to do paging, then I have to generate a nextToken value as well and include it in the output. I’ve done these changes for paging in [the GitHub repository](https://github.com/adrianhall/restaurant-reviews/) if you wish to take a look.

If you want to make the paging request for ElasticSearch work the same as for DynamoDB, then you need to convert the `nextToken` from a string to an integer. In velocity template language, this is done with a hack:

```
#set($Integer = 0)
#set($nextToken = $util.defaultIfNull($ctx.args.nextToken, "0"))
#set($from = $Integer.parseInt($nextToken))
```

You can combine the last two lines, but you must have the first `#set` to set up the ability to parse integers. This uses the Java `Integer.parseInt()` under the covers.

## Conclusion

At this point, I have an awesome GraphQL API implemented using AWS AppSync and with repeatable deployments. It supports both unauthenticated and authenticated connections (as long as the request is signed with SIGv4), and supports geospatial searches for locations, plus paged requests for my locations, reviews, and favorites.

There is still more to do here, and I will definitely be re-visiting this API in the future as I develop the apps for it.

You can find the completed API on [my GitHub repository](https://github.com/adrianhall/restaurant-reviews/).
