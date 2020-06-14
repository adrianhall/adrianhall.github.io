---
title: "How to deploy a GraphQL API on AWS with the Serverless Framework"
categories:
  - Cloud
tags:
  - GraphQL
  - "Serverless Framework"
  - Deloyment
---

In previous posts, we’ve explored how to deploy a GraphQL service based on [AWS AppSync] and Amazon DynamoDB using [AWS CloudFormation]. The articles reinforce how automatic and repeatable deployments are central to moving towards a DevOps mindset.

However, there are a few problems with the [AWS CloudFormation] template. The most notable issue is the inability to separate the schema and resolver mapping templates from the actual CloudFormation template. Overcoming this obstacle requires developers to embed these resources — and that makes it harder than necessary.

> AWS AppSync is a serverless backend-as-a-service for mobile and web applications. It’s a fully managed scalable platform that provides real-time data synchronization and offline capabilities for data driven applications. It leverages GraphQL, a powerful API query language that efficiently uses the network to reduce bandwidth requirements and network latencies.

There are two common deployment frameworks that can be used instead of CloudFormation — the [Serverless Framework](https://serverless.com), originally intended for deploying serverless applications based on AWS Lambda, and [Terraform](https://www.terraform.io/), which has the same aim. Both these tools aim to make infrastructure as maintainable as code, allowing you to check them in and test them.

## Getting started with Serverless Framework

In this article, I’m going to use the Serverless Framework to deploy the same GraphQL service that I deployed previously. To start, you need to have the Serverless Framework installed:

{% highlight bash %}
npm install -g serverless
{% endhighlight %}

This gives you two new commands: `serverless` and `sls — they` are equivalent to one another. Now, let’s create a new serverless project:

{% highlight bash %}
serverless create --template aws-nodejs --path sls-appsync-backend
cd sls-appsync-backend
npm init -y
git init
git add -A
git commit -m "Initial Commit"
{% endhighlight %}

The directory that is created contains two files — `handler.js` is the initial Lambda function for the project. We won’t be using it. The `serverless.yml` file contains the definition of the backend.

## Add plugins

One of the neat things about the Serverless Framework is the number of plugins that are available. You can create your own plugin and plenty of people have. You can see [a big list on GitHub](https://github.com/serverless/plugins).

To use a plugin, download it via npm and then add it to the serverless.yml file. In this project, I’m going to use just one plugin: [serverless-appsync-plugin](https://github.com/sid88in/serverless-appsync-plugin). This is a plugin for, you guessed it, configuring AWS AppSync, written by [Siddharth Gupta](https://twitter.com/sidg_sid).

To install, run the following (all on one line):

{% highlight bash %}
npm install -s serverless serverless-appsync-plugin
{% endhighlight %}

I also like to add a scripts section within the `package.json` file so that I don't have to remember how to deploy the service:

{% highlight json %}
"scripts": {
  "deploy": "./node_modules/.bin/serverless deploy -v"
}
{% endhighlight %}

Don’t deploy the service yet. We haven’t finished things! Edit the `serverless.yml` file and add the plugin to that:

{% highlight yaml %}
plugins:
  - serverless-appsync-plugin
{% endhighlight %}

You can put this new section right under the service name.

## Define dependent services

If you followed my prior article for using CloudFormation, you will know that I need a DynamoDB database, and Amazon Cognito user pool, and some IAM roles to hook them all together. These are placed in the `resources` section of the `serverless.yml` file and they look pretty much the same because they are the same.

Here is the list of resources you will need to define:

* An Amazon Cognito user pool
* A user pool app client
* The DynamoDB table
* A policy document for allowing access to the DynamoDB table
* An IAM role for implementing the policy document

I haven’t covered these here since I covered them in my prior article and they are identical in form.

There is a little complexity here, however. When I was doing a CloudFormation template, I could use operations like `!Join` and `!GetAtt`. These are not available with Serverless Framework. Instead, you have to use the JSON versions — mostly because Serverless Framework translates the configuration into a JSON CloudFormation file. That means using arrays and objects and the `Fn::` versions of the functions. As an example, the resource definition for the `AppSyncDynamoDBPolicy` became a bit cumbersome:

```
{ Fn::Join: [ '', [ { Fn::GetAtt: [ NotesTable, Arn ] }, '/*' ] ] }
```

This is bit incoherent and I hope that the Serverless Framework folks improve this so we can do something a little easier.

## Define the AWS AppSync schema and mapping files

This portion gets to the crux of the reason for switching to the Serverless Framework in the first place — we get to separate out the files that are used to define the AWS AppSync service. This means writing a schema and some mapping template files.

First, let’s look at the schema. This should be named `schema.graphql`. Yes — that specifically. Although you can specify the name of the file in the configuration file, I like the convention over configuration approach here.

The GraphQL schema is pulled from my CloudFormation template:

{% highlight graphql %}
type Note {
  NoteId: ID!
  title: String
  content: String
}
type PaginatedNotes {
  notes: [Note!]!
  nextToken: String
}
type Query {
  allNotes(limit: Int, nextToken: String): PaginatedNotes!
  getNote(NoteId: ID!): Note
}
type Mutation {
  saveNote(NoteId: ID!, title: String!, content: String!): Note
  deleteNote(NoteId: ID!): Note
}
type Schema {
  query: Query
  mutation: Mutation
}
{% endhighlight %}

Similarly, the mapping files must be located in a directory called `mapping-templates` relative to the `serverless.yml` file. Again, you can rename this directory, but why would you. The nice thing about this approach is that you can duplicate the mapping templates. For example, most of us have multiple response mapping templates that look like this:

```
$util.toJson($ctx.result)
```

That can become `common-response.vtl`. I like using the `.vtl` as an extension as it can enable some extra highlighting in editors. Visual Studio Code has a couple of potential syntax highlighters for VTL code. As with the resources, I’m not going to duplicate each and every mapping template here.

## Define the AWS AppSync API in serverless.yml

Now comes the interesting part — at least for me. That’s defining the AWS AppSync resource. It is not, as you might expect, included in the Resources section. Rather, it is included in a `custom` section.

Here is the example for my specific resource:

{% highlight yaml %}
custom:
  accountId: { Ref: AWS::AccountId }
  appSync:
    name: ${self:provider.apiname}
    authenticationType: AMAZON_COGNITO_USER_POOLS
    userPoolConfig:
      awsRegion: { Ref: AWS::Region }
      defaultAction: ALLOW
      userPoolId: { Ref: UserPool }
    schema: schema.graphql # In case you want to change it
    serviceRole: "AppSyncServiceRole"
    dataSources:
      - type: AMAZON_DYNAMODB
        name: NotesTableDS
        description: "DynamoDB Notes Table"
        config:
          tableName: { Ref: NotesTable }
          serviceRoleArn: { Fn::GetAtt: [ DynamoDBRole, Arn ] }
    mappingTemplates:
      - dataSource: NotesTableDS
        type: Query
        field: allNotes
        request: "allnotes-request.vtl"
        response: "allnotes-response.vtl"
      - dataSource: NotesTableDS
        type: Query
        field: getNote
        request: "getnote-request.vtl"
        response: "common-response.vtl"
      - dataSource: NotesTableDS
        type: Mutation
        field: saveNote
        request: "savenote-request.vtl"
        response: "common-response.vtl"
      - dataSource: NotesTableDS
        type: Mutation
        field: deleteNote
        request: "deletenote-request.vtl"
        response: "common-response.vtl"
{% endhighlight %}

Each query and mutation has an entry in the `mappingTemplates`. You can also provide mapping templates for individual fields and for subscriptions if you so desire. Note how I use the `common-response.vtl` in three out of four of the resolvers — code re-use is a beautiful thing.

The other parts of the AWS AppSync configuration — authentication, schema, and data sources — get their own sections. Where necessary, I reference the appropriate resources from the `resources` section.

## Deploy your API

To deploy your API, run the following:

{% highlight bash %}
serverless deploy -v
{% endhighlight %}

The Serverless Framework will get to work. First, it converts your serverless.yml and files to a CloudFormation template, then it will create an S3 bucket to handle the deployment, and finally it will deploy the CloudFormation template for you. You can check out the contents of the `.serverless` directory to see this in action.

If you’ve made an error (I made a few dozen while building this), then you can adjust the files and run `serverless deploy -v` again to correct the issue. The Serverless Framework will figure it all out for you.

How do you figure out the error? Well, I go to the AWS CloudFormation console, click the CloudFormation stack I am deploying, and then view the failure event details. There is usually something in there that indicates which resource failed and why.

Normally, there will be multiple failures, but you will have one that is an actual error in the template. Then you can go to the resource within your `serverless.yml` file and try and figure out what is wrong with it.

Sometimes, the differences between the YAML for Serverless Framework and for CloudFormation will be surprising. I find the majority of my errors are in specifying IAM roles and policies, and the error messages don’t necessarily point to the exact error here — although they do point to the right resources.

The good news? We are using CloudFormation underneath. That means if there is an error, the whole stack is rolled back, allowing you to correct the error and re-try the deployment.

## Remove your API

Removing the API can be done via the serverless command as well:

{% highlight bash %}
serverless remove -v
{% endhighlight %}

This, again, does CloudFormation underneath. That means that resources that contain data (such as S3 buckets and DynamoDB tables) won’t be deleted through this mechanism. You will have to manually delete those resources.

## Conclusion

Is Serverless Framework “better” than straight-up CloudFormation? Well, the answer is “it depends”. I like the separation of the AWS AppSync schema and mapping templates into their own components.

I also daresay that if I added in AWS Lambda resolvers, the Serverless Framework would definitely be better. However, I had problems with the syntax of the resources section. It just wasn’t as clean as the CloudFormation version, partly because of the way that things were constructed.

There are also other concerns. I am hardly ever deploying a single API. It’s more likely to be part of a native app or web app. In this case, I would want the Serverless Framework to be part of a deployment step that also extracts the appropriate configuration files that my front end application would use. I’m not saying it isn’t doable, but it is definitely harder. I also have to learn yet another syntax with yet another set of quirks.

That being said, the splitting of concerns and the ability to check your infrastructure definition into a source code repository are definitely on the plus side. Whether you use CloudFormation, Serverless Framework, Terraform, the awsmobile CLI or your own special deployment scripts, automation is the key to trouble-free deployments.

{% include links.md %}
