---
title: "How developers can authenticate and authorize users with AWS AppSync"
categories:
  - Cloud
tags:
  - aws_appsync
  - graphql
---

> AWS AppSync provides four distinct methods of authorizing users to optimize and restrict data being transferred

[AWS AppSync] is a managed [GraphQL] data service that supports offline and real-time scenarios. The service allows the developer to optimize the data transfer between client and server.

Any non-trivial application will need to authenticate users. It’s the only way to identify a distinct real person using the application. That association is required for private data storage and compliance reasons — for example, GDPR includes [a right of erasure](https://gdpr-info.eu/art-17-gdpr/).

Developers will also have to manage data that is associated with an individual, group or role and operations with that data must be restricted — that’s authorization. In short, you need both authentication and authorization.

AWS AppSync provides four distinct methods of authorizing users:

* API key
* OpenID Connect (OIDC)
* Amazon Cognito user pools
* AWS Identity and Access Management (IAM)

In addition, each option has several parameters that you can set to ensure that you get the right authorization for each query and mutation that an app can execute. So, which method do you use?

## API Key

You shouldn’t use API keys for user authorization in a production app because there’s no concept of identity. Under most circumstances, developers should only leverage an API key when you’re building a read-only public-facing application — or in the prototyping phase of your application development cycle. After prototyping, you can switch to a different authorization method.

You can use any client that supports HTTPS transport with an API key. Just add an `X-API-KEY` header to each request. The value of the header should be the API key. When you choose this option, the API key is listed within the AWS AppSync settings.

## OpenID Connect (OIDC)

Use OIDC if you need to integrate with a pre-existing identity provider and don’t want to do federation — _more on that later_.

OIDC allows you to work with users by passing a JSON Web Token (JWT) token as an authorization header in the HTTPS request. You can use any GraphQL client that supports adding a header to the HTTPS request.

You can be more granular with authorization by filtering on the claims in the token (which are available through `$context.identity.claims`) in the VTL-based resolver templates. However, there’s no real notion of “groups” or “roles” unless they’re added as claims by the third-party provider.

If you want to store data that’s associated with the user, ensure that you choose a claim that doesn’t change when the user changes their information. Usually, `$context.identity.sub` is a good choice, but that’s not a guarantee.

## Amazon Cognito user pools

Using an Amazon Cognito user pool is perhaps the most flexible option, so I recommend using this option if you have no other authentication source and you’re setting up everything from scratch.

A user pool is an OIDC-compliant authentication provider with some additional benefits — such as group functionality. So you can use any GraphQL client that allows you to alter the headers of the request. Add a bearer authorization header to each request to authenticate the user.

AWS AppSync has built-in support for the specifics of user pools. The main thing that you can do with user pools that you can’t do with generic OIDC authentication is to organize users into groups.

User pools support direct federation with Facebook and Google authentication providers. This allows you to do single sign-on with those environments. For federating with other providers, check out AWS IAM as a source.

Integrating your app with a user pool is easy. The AWS Mobile SDK provides pre-built screens for iOS (Objective-C and Swift) and Android (Java and Kotlin). AWS Amplify provides support for React Native and single-page applications that are written with React and Angular. There are also low-level interfaces for most environments if you want to generate your own authentication screens.

I’ll be talking more about user pools a little later — _so stay tuned_!

## Identity Access Management (IAM)

The final authorization method that you can use is IAM. This is a good option if you have an existing Amazon Cognito identity pool (identity pools are also known as federated identities), or if you’ve created an access key and secret key for the application. The access permissions are handled by IAM roles.

An identity pool can federate with a wide variety of authentication providers — including Facebook, Google, user pools, and any other OIDC or SAML provider, including Active Directory Federated Services (AD FS). Using IAM is also the only current method that allows both unauthenticated and authenticated requests within the same API.

To use IAM, the request must be signed with AWS Signature Version 4. Most clients don’t support AWS Signature Version 4 out of the box. That limits you to the Apollo client with the AWS AppSync transport, the AWS Amplify client, and the AWS Mobile SDK for iOS and Android.

## Options for Amazon Cognito user pools

You can see why I prefer using an Amazon Cognito user pool — it’s the most flexible of the options I’ve described. Developers can use any GraphQL client, provide group-based authorization rules with fine-grained access to claims from the directory, and federate with Facebook and Google. There’s a lot of reasons to recommend this option.

Authorization is split into three distinct phases:

1. First, the schema is consulted for `@aws_auth` directives. If the user is a member of a listed group, then the operation is allowed.
2. If there’s no `@aws_auth` directive, then the global setting (`ALLOW` or `DENY`) is used as the permission.
3. You can also specify (fine-grained access control) rules within the VTL request or response resolver to return `$util.unauthorized()` for more specific authorization rules.

Most APIs use `ALLOW` as the default action. That means that a user can access the API unless they’re explicitly prevented from doing so by an `@aws_auth` directive.

The alternative is `DENY`, which means that you must be explicitly allowed to do something by using an `@aws_auth` directive. (This really means that every single query and mutation must have an `@aws_auth` directive, and every user must belong to a group.)

Let’s use an example of writing a blogging platform. We might define a blog as follows:

{% highlight graphql %}
type Post {
  postId: ID!
  author: String!
  title: String!
  content: String!
  category: String!
}
{% endhighlight %}

We’ll want public access to the blog, sol let’s set the default permission to `ALLOW`. However, we also want special permissions on the `addPost()` mutation. Only members of the `editors` group, which we’ve defined inside our user pool, can post things:

{% highlight graphql %}
type Mutation {
  addPost(title: String!, content: String!, category: String!): Post
  @aws_auth(cognito_groups: ["editors"])
}
{% endhighlight %}

In addition, we’ll want a separate setting so that only users in the `announcements` group, another group we’ve defined inside our user pool, can post to the `Announcements` category. This is done inside the request resolver:

{% highlight text %}
#foreach($group in $ctx.identity.claims.get("cognito:groups"))
#if($group == "announcements")
#set($inAnnouncementsGroup = true)
#end
#end
#if(($ctx.args.category=="Announcement" && $inAnnouncementsGroup == true) || ($ctx.args.category != "Announcement"))
{
  “version”: “2017–02–28”,
  “operation”: “PutItem”,
  “key”: {
    “postId”: { “S” : “$utils.autoId()” }
  },
  “attributeValues”: {
    “owner”: { “S” : “${ctx.identity.sub}” },
    “title” : { “S” : “${ctx.args.title}” },
    “content”: { “S”: “${ctx.args.content}” },
    “category”: { “S” : “${ctx.args.category}” }
  }
}
#else
$utils.unauthorized()
#endif
{% endhighlight %}

The first few lines determine if the user is within the right group. Developers can do any checks we want here. For example, these checks might be verifying that the user is within the corporate network, or that the user has a specific claim within the directory.

The next step is to decide if the user is allowed to do the query. If they are allowed, then perform the operation. Otherwise, return unauthorized.

## Filtering on app clients

Both the OIDC provider and the user pools provider have the option to specify an AppId client regular expression. This allows you to allow (or block) requests to the API based on the app client ID.

The app client is defined within the OIDC provider or the Amazon Cognito console. The easiest way to use it is to provide an OR’ed list of allowed AppId’s — because they don’t tend to be easily digested by a regular expression.

Why would you even use this? Well, let’s say you have a set of clients for mobile apps (one for Android, one for iOS, one for web, and so on) and a set of clients for server-side operations. You might want to prevent the clients for server-side operations from using the GraphQL API — following the principal of providing the least access. You might also want to deprecate clients over time. You can control this by filtering as well.

## Summary

AWS AppSync offers authentication and authorization options that provide a lot of flexibility. The AWS AppSync team is constantly evolving the authorization capabilities, so it’s worth looking at the latest options.

I highly recommend reading the [authorization use cases documentation](https://docs.aws.amazon.com/appsync/latest/devguide/security-authorization-use-cases.html) in the [AWS AppSync Developer Guide](https://docs.aws.amazon.com/appsync/latest/devguide/welcome.html) next, as that document shows off a lot of the power of the AWS AppSync service.

{% include links.md %}
