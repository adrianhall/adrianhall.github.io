---
title: "Building a GraphQL API by Example: Restaurant Reviews (part 2: The Backend)"
categories:
  - Cloud
tags:
  - GraphQL
  - "zCloud:AWS"
---

I'm currently in the middle of building a restaurant review API driven by GraphQL. So far, I've [looked at the needs of the front end and developed the schema]({% post_url 2018-11-15-restaurant-reviews %}). Now, I need to take a look at the back end. I'm building the API within the AWS cloud. How do I store the data?

Let's take a quick look at the schema again:

```graphql
input PagingRequest {
    limit: Int
    nextToken: String
}

type User {
    id: ID!
    name: String
    locations(paging: PagingRequest): LocationPagingConnection
    reviews(paging: PagingRequest): ReviewPagingConnection
    favorites(paging: PagingRequest): LocationPagingConnection
}

type GPS {
    longitude: Int
    latitude: Int
}

input GPSInput {
    longitude: Int
    latitude: Int
}

type Address {
    street: String
    city: String
    state: String
    zipcode: String
}

type Location {
    id: ID!
    owner: User!
    name: String!
    gps: GPS
    address: Address
    phone: String
    email: String
    favoriteCount: Int
    averageRating: Float
    reviews(paging: PagingRequest): ReviewPagingConnection
}

type Review {
    id: ID!
    user: User!
    location: Location!
    content: String
    rating: Int
}

type UserPagingConnection {
    items: [User]
    nextToken: String
}

type LocationPagingConnection {
    items: [Location]
    nextToken: String
}

type ReviewPagingConnection {
    items: [Review]
    nextToken: String
}

input GPSQueryInput {
    gps: GPSInput
    radius: Float
}

input AddressInput {
    street: String
    city: String
    state: String
    zipcode: String
}

input LocationInput {
    name: String
    address: AddressInput
    phone: String
    email: String
}

input ReviewInput {
    content: String
    rating: Int
}

type Query {
    me: User!
    findLocation(byGPS: GPSQueryInput, byAddress: AddressInput): LocationPagingConnection
}

type Mutation {
    addLocation(location: LocationInput): Location
    addReview(locationId: ID!, review: ReviewInput): Review
    markFavorite(locationId: ID!): Location
}

type Subscription {
    location(locationId: ID!): Location
    @aws_subscribe(mutations: [ "addReview", "markFavorite" ])
    reviews(locationId: ID!): Review
    @aws_subscribe(mutations: [ "addReview" ])
}

schema {
    query: Query
    mutation: Mutation
    subscription: Subscription
}
```

Data backend decisions tend to be made based on the queries that will be performed. Lots of different database technologies can support storage. However, not a lot of database technologies can do everything when we look at the queries.

When looking at these queries, most of the queries are straight forward and can probably be handled by any of the data stores. However, there are some that I have explicit concerns about:

* _findLocation(byGPS)_ requires us to do geolocation queries within a radius.
* _location(locationId)_ requires us to return a count of favorites and an average of the review ratings.
* There are several queries that return paged responses.

AWS AppSync (my favorite GraphQL data layer) supports DynamoDB (a NoSQL service), ElasticSearch Service, and Aurora Serverless. 

When deciding on a backend solution:

1. Ensure your choice supports your queries and mutations.
2. Don't be afraid to use multiple data sources for different parts of your schema.

Let's investigate some choices:

## Amazon DyanmoDB

I love DynamoDB. It's a serverless key-value store that allows you to control the scaling (and hence the costs). However, it isn't always a good choice. Specifically, it's particularly bad if you have to do aggregations across the entire data set. Unfortunately, that means I can't calculate the ratings average on the fly. In addition, I can't do geospatial search. That makes two items on my list of queries problematic.

## Amazon DynamoDB + ElasticSearch Service

There is an alternative to using DynamoDB directly. You can store the data within DynamoDB (so all mutations happen against a DynamoDB instance) and then send that data to an ElasticSearch Service instance. All the queries hit the ElasticSearch Service instance instead. ElasticSearch Service can handle geospatial searches, counts and average aggregation. This means I can do all the mutations and even some of the queries via DynamoDB, then switch over to ElasticSearch Service where required.

In addition, I can see the need to do full-text search at some time in the future. One of the things I try to do is predict the sorts of queries that might come along in the future as I decide on the backends I am going to implement.

However, we are going towards two different stores which we need to synchronize via a Lambda function. This isn't hard to do, but does increase the complexity. In addition, ElasticSearch Service is not "serverless", so we have to deal with scaling the search service using configuration:

![]({{ site.baseurl }}/assets/images/2018-11-30-picture1.png)

## Aurora Serverless

The third option is to use a SQL database based on Aurora Serverless. This is new and is definitely worthwhile considering when you have relational data. I can create a Locations table and a Reviews table and then provide relations between them. Since SQL provides the ability to store GPS coordinates as points, I can do geospatial searches easily. I can also do average rating and count of reviews as part of the same SQL query.

## Which should I choose?

I have two viable options here - and it's normal for there to be multiple viable options. For this project, I'm going to implement the DynamoDB + ElasticSearch Service, mostly because I can see full-text search in my future which is more appropriately handled by ElasticSearch Service. I will implement each record (locations and reviews) within a single DynamoDB table with an ID and a Type (LOCATION or REVIEW). This will then be transferred to the ElasticSearch Service for searching. The majority of the queries will be handled by the ElasticSearch Service, although there are some queries (for example, the paging query for reviews at a location) that will be accomplished with a DynamoDB query.

## Implementing the Data Storage Layer

Now that I've decided on an architecture for my data storage layer, all that remains is to set it up. There are three basic ways of configuring the backend if you want to remain sane. 

* The [Amplify CLI](https://amplify.aws) will configure your GraphQL API easily by using model transforms. However, we are doing a completely custom GraphQL API which is not supported at this time.
* [CloudFormation](https://aws.amazon.com/cloudformation/) allows you to configure all the resources in the backend, but I want to configure the front end too.
* [Serverless Framework](https://serverless.com/framework/) (with the appsync-serverless-plugin and aws-amplify-serverless-plugin) allows me to do all the configuration using CloudFormation but also has an easy way of writing out the front end configuration files.

Short version. Use Amplify CLI if you can as it automates a lot of things, simplifies what you have to write and generates code so that you speed up development. Switch to the Serverless Framework when the Amplify CLI doesn't do all the things you need for maximum flexibility. Switching to the Serverless Framework also ramps up the complexity you have to deal with. You will still need to know CloudFormation, but a lot of the complexity of dealing with embedded files is taken care of for you.

> What about SAM CLI, Terraform, or any of the other configuration tools? These don't support both front end and back end configuration of AWS AppSync at the time of writing, so I'm not using them.

I've already got a Serverless Framework configuration [started for my backend](https://github.com/adrianhall/restaurant-reviews), which includes the `schema.graphql` file from my last blog. All that is required is that I add the backend configuration to my `serverless.yml` file. This requires the following resources:

* A DynamoDB table
* An ElasticSearch domain
* A Lambda function to transfer data from the DynamoDB table to the ElasticSearch domain
* An IAM Role to provide access restrictions to the Lambda function

The DynamoDB table is simple enough as I've written these plenty of times before:

```yaml
DynamoDBTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: ${self:custom.api}
    KeySchema:
      - AttributeName: id
        KeyType: HASH
      - AttributeName: typeName
        KeyType: RANGE
    AttributeDefinitions:
      - AttributeName: id
        AttributeType: S
      - AttributeName: typeName
        AttributeType: S
    ProvisionedThroughput:
      ReadCapacityUnits: ${self:custom.ddb_readIOPS}
      WriteCapacityUnits: ${self:custom.ddb_writeIOPS}
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
```

This is placed in the `Resources` section of the `serverless.yml` file. The only new thing here is the `StreamSpecification`. This sets up the data stream such that when data is added, updated, or removed from DynamoDB, the change is also sent to the stream.

The ElasticSearch domain is similarly boiler plate:

```yaml
ElasticSearchDomain:
  Type: AWS::Elasticsearch::Domain
  Properties:
    DomainName: ${self:custom.es_domain}
    ElasticsearchVersion: "6.2"
    ElasticsearchClusterConfig:
      ZoneAwarenessEnabled: false
      InstanceCount: ${self:custom.es_instanceCount}
      InstanceType: ${self:custom.es_instanceType}
    EBSOptions:
      EBSEnabled: true
      VolumeType: "gp2"
      VolumeSize: ${self:custom.es_ebsVolumeGB}
```

Note that there is no access policy here, despite the CloudFormation template allowing one. My access to the ElasticSearch domain will be programmatic via AWS AppSync or AWS Lambda, so I need to be a bit more prescriptive to honor the rule of least access possible.

I've added a whole bunch of configuration elements within the custom section of the serverless.yml file so that I don't have to look through a massive YAML file to find out where to change the scaling options:

```yaml
custom:
  # The base name of the API for resource generation - can't include dashes
  # [a-zA-Z0-9]+ only
  api: ${self:provider.stage}RestaurantReviews
  # ES Domain, must match [a-z][a-z0-9\-]+
  es_domain: ${self:provider.stage}-restaurant-reviews
  # The number of instances to launch into the ElasticSearch domain
  es_instanceCount: 1
  # The type of instance to launch into the ElasticSearch domain
  es_instanceType: "t2.small.elasticsearch"
  # The size in GB of the EBS volumes that contain the data
  es_ebsVolumeGB: 20
  # The number of read IOPS the DynamoDB table should support.
  ddb_readIOPS: 5
  # The number of write IOPS the DynamoDB table should support.
  ddb_writeIOPS: 5
```

The IAM role for the Lambda function should allow the function to receive triggers from DynamoDB and then write to ElasticSearch. It also needs to be able to write to Cloudwatch logs for debugging purposes:

```yaml
ElasticSearchStreamingLambdaIAMRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: ${self:custom.api}-ESStreamingLambdaRole
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal:
            Service: "lambda.amazonaws.com"
          Action: "sts:AssumeRole"
    Policies:
      - PolicyName: ElasticSearchAccess
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Action:
                - "es:ESHttpPost"
              Effect: Allow
              Resource:
                - "arn:aws:es:#{AWS::Region}:#{AWS::AccountId}:domain/${self:custom.es_domain}/_bulk"
      - PolicyName: DynamoDBStreamAccess
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Action:
                - "dynamodb:DescribeStream"
                - "dynamodb:GetRecords"
                - "dynamodb:GetShardIterator"
                - "dynamodb:ListStreams"
              Effect: Allow
              Resource:
                - { Fn::GetAtt: [ DynamoDBTable, StreamArn ]}
      - PolicyName: CloudWatchLogsAccess
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Action:
                - "logs:CreateLogGroup"
                - "logs:CreateLogStream"
                - "logs:PutLogEvents"
              Effect: Allow
              Resource:
                - "arn:aws:logs:#{AWS::Region}:#{AWS::AccountId}:*"
```

Finally, here is the configuration of the Lambda function:

```yaml
functions:
  dynamodb_stream:
    handler: elasticsearch.lambda_handler
    name: ${self:custom.api}-dynamodb_stream_handler
    description: Stream data from DynamoDB to ElasticSearch
    runtime: python3.6
    memorySize: 128
    role: ElasticSearchStreamingLambdaIAMRole
    environment:
      ES_ENDPOINT: { Fn::GetAtt: [ ElasticSearchDomain, DomainEndpoint ]}
      DEBUG: 1
    events:
      - stream:
          type: dynamodb
          arn: { Fn::GetAtt: [ DynamoDBTable, StreamArn ]}
```

I copied the function directly from the Amplify CLI, and just slightly adjusted it so I could pass the `StreamArn` in directly instead of having to put the https:// in front of it. This Lambda function is triggered via the DynamoDB stream. When there is a stream request, the script will copy the changes to the ES domain. You can find the code for the function on [my GitHub repository](https://github.com/adrianhall/restaurant-reviews/tree/p2).

> If you think this is complicated, then check out the Amplify CLI. It does all this configuration for you so you don't have to worry about it!

## Testing the data transfer

I never leave this step without ensuring that basic data transfer is happening. However, that requires set up.

1. Log on to the [ElasticSearch Service Console](https://console.aws.amazon.com/es/home).
2. Select your ElasticSearch domain.
3. Click **Modify Access Policy**.
4. In the drop-down, select **Allow open access to the domain**.
5. Click **Submit**.
6. Wait for the change to be processed (it takes 1–2 minutes).

Anyone who has the endpoint information can now access your ES domain. This is dangerous, so only do this for testing and remove it once testing is done. For you, this means you can run Kibana:

1. Go back to the Dashboard in the ElasticSearch Service Console.
2. Select your ElasticSearch domain.
3. Click on the **Kibana** link.
4. Click on **Dev Tools**.
5. If necessary, click **Get Started**.

At this point, you have a query window, with a "query for all data" specification already loaded. Click on the Play button to see the results (it should result in no data).

Now, let's test the data transfer.

1. Go to the [DynamoDB Console](https://console.aws.amazon.com/dynamodb/home).
2. Click **Tables**, then select your table.
3. Select the **Items** tab.
4. Click **Create Item**.
5. Fill in the fields (id and typeName), then click **Save**.

After a couple of seconds, the data should have appeared in the ES domain. Go back to your Kibana tab and run the query again. This time, you should get data:

![]({{ site.baseurl }}/assets/images/2018-11-30-picture2.png)

If you delete or update the record within the DynamoDB console, the changes should be reflected in the Kibana query.

## Next Steps

I still don't have a functional API. However, I have all both the schema for the API and the backend data storage handled. In the next blog post, I'll cover writing the request and response mapping templates for the resolvers using AWS AppSync.

Until then, you can find the Serverless Framework configuration on [my GitHub repository](https://github.com/adrianhall/restaurant-reviews/tree/p2).
