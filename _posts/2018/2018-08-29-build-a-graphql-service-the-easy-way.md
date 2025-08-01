---
title: "Build a GraphQL Service the easy way with AWS Amplify Model Transforms"
categories:
  - Cloud
tags:
  - aws_appsync
  - graphql
---

Creating a functional GraphQL API is hard. You have to create a GraphQL schema, decide on authentication and database structures, implement the schema in a GraphQL service, wire up the authentication, hook up the database sources, ensure the whole thing is scalable, worry about logging and monitoring, and then write your app.

[AWS AppSync] helps you with everything except the GraphQL schema and the app. Now, [AWS Amplify] is helping you with the GraphQL schema by introducing model transforms. This is an open-source library that transforms an annotated GraphQL schema into a real GraphQL schema that you can deploy (technically, the schema is embedded in an [AWS CloudFormation]  [template](https://aws.amazon.com/cloudformation/aws-cloudformation-templates/) that can be deployed on AWS). If you are using the AWS Amplify CLI (and you should — it makes the process much easier), it will also deploy all the resources necessary to implement the GraphQL schema on AWS AppSync and wire them up with appropriate VTL mapping templates to properly do the operations.

Let’s take a little example. I use my notes app as my go-to teaching tool. It uses one table — a list of notes. I can represent this in standard GraphQL schema language like this:

{% highlight graphql %}
type Note {
  id: ID!
  title: String!
  content: String!
}
type Query {
  getNote(id: ID!): Note
  listNotes: [Note]
}
type Mutation {
  createNote(title: String!, content: String!): Note
  updateNote(id: ID!, title: String!, content: String!): Note
  deleteNote(id: ID!): Note
}
schema {
  query Query
  mutation Mutation
}
{% endhighlight %}

Of course, this isn’t “best practices”. It’s probably very similar to the first GraphQL schema just about everyone who is learning the schema language writes. We do basic CRUD operations and don’t worry about searching, sorting, pagination, input types, or subscriptions. Those things can be added on later.

With an annotated schema, we can write the following:

{% highlight graphql %}
type Note @model {
  id: ID!
  title: String!
  content: String!
}
{% endhighlight %}

Yep — that’s it. Just five lines. What happens when you run it through the model transforms library (and deploy it via the AWS Amplify CLI) is that this is expanded. You get a CRUD API with paging and filtering built-in; input types are created for each of the mutations and subscriptions are set up on each mutation; the backing store in a DynamoDB database. Five lines turns into over 100 lines of schema that will take care of most of the requirements of your typical app.

The `@model` is an annotation that tells the model transform library to do something. There are a big list of them, but basically, they fall into three camps:

* Backing or data store directives
* Authorization directives
* Capabilities directives

## Backing Data Store

I’ve already introduced you to one data store directive in my example above. The `@model` directive stores the data in a table dedicated to the type within [Amazon DynamoDB]. This is good for basic searches and filtering, but you may want better searchability of your data. For example, you might want to do geo-lookups, or faceted search. For this sort of functionality, it would be a good idea to stream the data from DynamoDB to an ElasticSearch Service. To do that, add the @searchable directive to the type:

{% highlight graphql %}
type Note @model @searchable {
  id: ID!
  title: String!
  content: String!
}
{% endhighlight %}

Underneath the covers, [AWS Amplify] deploys a DynamoDB table and an ElasticSearch Service instance with an [AWS Lambda] function that streams data from DynamoDB to ElasticSearch. All the queries will hit the ElasticSearch Service and all the mutations will be sent to DynamoDB.

## Authorization

Before model transforms, you needed to adjust the VTL-based mapping templates to properly implement authorization within the API. The model transforms library makes authorization a breeze. Let’s say I wanted to change my note into a multi-tenant service. Here is how I would do it:

{% highlight graphql %}
type Note
    @model
    @auth(rules: [{ allow: owner }])
{
  id: ID!
  title: String!
  content: String!
}
{% endhighlight %}

The `@auth` directive provides rules to say who can access the specific type operations. These can get quite complex. Note that the rules is a JSON array of individual rules. Each rule can have several components, the only one of which is required is the allow field.

Let’s take a slightly more complex version of this. Let’s say you wanted to allow the owner to create, update, and delete their own notes, but you wanted the owner and a group of people called “managers” to read the notes:

{% highlight graphql %}
type Note
  @model
  @auth(rules: [
    { allow: owner },
    { allow: groups, groups: ["managers"], queries: ["get","list"] }
  ])
{
    id: ID!
    title: String!
    content: String!
}
{% endhighlight %}

If any auth rule matches, then the operation is allowed. Here is a list of all the things you can currently set:

* The `allow` field is required and can be “owner” or “groups”
* The `ownerField` is the field where the owner of the record is stored. This defaults to the field name `owner`.
* The `identityField` is the field within the `$context.identity` object within the VTL that you store in the ownerField to recognize the owner. It defaults to `username`.
* The `groupsField` is the field within record that provides the group information that is compared.
* The `groups` field is the list of groups to allow when the `allow` field is set to `groups`. This field can be a string or an array of strings if you want to represent multiple groups.
* The `mutations` field is the list of mutations to allow. This field is always an array and can consist of zero or more of “create”, “update” or “delete”.
* The `queries` field is the list of queries to allow. This field is (as with the mutations), an array and consists of zero or more of “get” and “list”.

If you don’t specify any queries or mutations, then it assumes all of the available operations are handled by the rule. If even one query or mutation is listed, then it assumes the others are not allowed by the rule.

I fully expect this area to get much richer as time goes by to support more and more authorization scenarios.

## Capabilities

To show off capabilities, let’s take a different model. Let’s say I have a blog application where multiple people can have a blog. Each blog can have a number of posts, and each post can have a number of comments. This may seem a little contrived, but relationships between types is common. Most applications will implement this with a SQL database and represent the types as individual tables that can be joined.

With GraphQL, you get cascading resolvers instead, so you don’t need to program in complex SQL queries to manage this for you. However, you need to model the data. For this, you can use the `@connection` directive. Let’s take a look at the blog example:

{% highlight graphql %}
type Blog @model
  @auth([
    {allow: groups, groups:"owners", mutations["create","update"}]},
    {allow: groups, groups:"everyone", queries:["get","list"]}
  ])
{
  id: ID!
  title: String!
  posts: [Post] @connection
}
type Post @model @searchable
  @auth([
    {allow: owner },
    {allow: groups, groups:"everyone", queries:["get","list"]}
  ])
{
  id: ID!
  title: String!
  content: String!
  comments: [Comment] @connection
}
type Comment
  @model
  @auth([
    {allow: owner },
    {allow: groups, groups:"everyone", queries:["get","list"]}
  ])
{
  id: ID!
  title: String!
  content: String!
}
{% endhighlight %}

The `@connection` supports both one-to-one and one-to-many relationships. There is a little bit more to do if you want the bi-directional connections to provide reverse linkage. For example, if you wanted to say “give me all the posts for a particular blog”, you may want the bi-directional linkage. You can do this by naming the connection:

{% highlight graphql %}
type Blog @model
  @auth([
    {allow: groups, groups:"owners", mutations["create","update"}]},
    {allow: groups, groups:"everyone", queries:["get","list"]}
  ])
{
  id: ID!
  title: String!
  posts: [Post] @connection(name: "BlogPosts")
}
type Post @model @searchable
  @auth([
    {allow: owner },
    {allow: groups, groups:"everyone", queries:["get","list"]}
  ])
{
  id: ID!
  title: String!
  content: String!
  blog: Blog @connection(name: "BlogPosts")
  comments: [Comment] @connection
}
{% endhighlight %}

Use the same name in both sides of the connection (one in the `Blog` type and one in the `Post` type).

Another good capability to add is versioning. This is especially useful when you are using the interface for an fully offline scenario when you need object versioning and conflict detection. In this scenario, when you send an update, you must also send a version field. This says “I’m updating record with id X and version Y”. If the current version doesn’t match Y, then a conflict response is produced. You can use this on the client to produce the correct dialog. Here is my original Notes type with versioning:

{% highlight graphql %}
type Notes @model @versioned {
  id: ID!
  title: String!
  content: String!
}
{% endhighlight %}

The current version is stored in a field called `version`. When you send an update or delete, send an `expectedVersion` field as well to handle conflict resolution.

Note that most of the directives (`@connection`, `@auth`, `@versioned`, and `@searchable`) currently rely on DynamoDB as the backing store, so they require the `@model` directive to be added to the type.

You can use the GraphQL transforms library independently of the AWS Amplify CLI as well. It produces a CloudFormation template that has the schema, resolver mapping templates, and resource descriptions that are needed to implement the transformed GraphQL API in [AWS AppSync]. Check out [the documentation](https://github.com/aws-amplify/amplify-cli/tree/master/packages/graphql-transformer-core) if you want to try that out.

## Conclusion

[AWS Amplify] is all about making you more productive as a front-end developer, and the GraphQL model transforms is one step in that process. Check out the other components — the AWS Amplify CLI for easily deploying web and mobile serverless backends and the AWS Amplify library for JavaScript to easily use the serverless backend you have set up. GraphQL has a relatively larger learning curve (compared to, say, REST), but it comes with a lot of power. Model transforms enables you to short circuit that learning curve so that you can get on with writing your app.

{% include links.md %}
