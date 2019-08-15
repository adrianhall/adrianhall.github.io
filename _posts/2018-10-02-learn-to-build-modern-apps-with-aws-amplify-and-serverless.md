---
title: "Learn to build mobile and web apps with AWS Amplify and Serverless Framework"
categories:
  - Mobile
tags:
  - Deployment
  - "Serverless Framework"
---

There has always been a bit of a problem between deploying your backend and then integrating that same backend with your front end code. The front end needs to know where the back end is located. The traditional approach is to create a configuration file for the purpose. But how?

The AWS Mobile team created the awsmobile CLI and followed that with the [AWS Amplify] CLI. These tools are great for new projects. They provide for best practices deployment capabilities on the backend — and automatic maintenance of the required configuration file on the front end.

## What about existing projects?

Let’s say you have a nice deployment of a backend on AWS with the [Serverless Framework](https://serverless.com/). You’ve may have configured Amazon Cognito, plus an Amazon DynamoDB table connected together with some AWS Lambda functions and an AWS AppSync API layer. Things are good. Now you want to create some mobile apps to leverage this infrastructure.

In the old world, you would create an `aws-exports.js` file for your web and React Native projects, or an `awsconfiguration.json` file for your Android and iOS native projects — probably by hand. The alternative would be to create a constants file with the appropriate values for initializing connectivity to your backend. Either way, it’s a laborious process.

That is why I created the [aws-amplify-serverless-plugin](https://www.npmjs.com/package/aws-amplify-serverless-plugin). This is a plugin for the Serverless Framework that takes the resources that you have deployed and creates the appropriate configuration file(s) for you.

## Let's look at an example

To get started with a simple example, let’s say we have a GitHub repository with four directories at the top level:

* `backend` contains my Serverless Framework files, including the `serverless.yml` file, GraphQL schema and mapping templates.
* `android` contains my typical Android project.
* `ios` contains my typical iOS project.
* `web` contains my web project that I will deploy to an S3 bucket that is specified in the `serverless.yml` backend configuration.

To deploy, we just type:

```
(cd backend && sls deploy -v)
```

Then we’ll build each of the clients, maybe deploying the web like this:

```
(cd android && ./gradlew build)
(cd ios && ./build.sh)
(cd backend && sls clinet deploy -v)
```

It’s a fairly normal build cycle. Now, let’s update the project to include the `aws-amplify-serverless-plugin`. First, we’ll need to install the plugin:

{% highlight yaml %}
plugins:
  - aws-amplify-serverless-plugin
{% endhighlight %}

Then, add a custom section to configure the plugin:

{% highlight yaml %}
custom:
  amplify:
    - filename: ../android/app/src/main/res/raw/awsconfiguration.json
      type: native
      appClient: AndroidUserPoolClient
    - filename: ../ios/App/awsconfiguration.json
      type: native
      appClient: iOSUserPoolClient
    - filename: ../web/src/aws-exports.js
      type: javascript
      appClient: WebUserPoolClient
{% endhighlight %}

Each appClient entry is an `AWS::Cognito::UserPoolAppClient` resource listed in the `resources` section of your `serverless.yml` file.

Now, run the deployment using `sls deploy -v`.

At the end of the deployment, you should see the three files written out. Android, React Native, and web clients can use the files directly. For iOS, you will have to add the `awsconfiguration.json` file to your project.  Ensure you do not check the “Copy files” box when adding the file to your project. You want any updates to be automatically included in your project.

## How does it work?

The Serverless Framework turns your configuration into a CloudFormation template. The plugin cycles through all the resources that are a part of the CloudFormation stack that has been deployed, and queries those resources for settings that are relevant to a mobile or web client. Once it has all the settings, it then turns the resulting configuration into the appropriate file format.

## Supported Features

The `aws-amplify-serverless-plugin` supports the following features

* [AWS AppSync] (including generation via the excellent `serverless-appsync-plugin` as well as through the Resources section)
* Amazon Cognito Federated Identities
* Amazon Cognito User Pools — you can specify a different user pool app client for each client.
* Amazon S3 buckets — the first listed S3 bucket (aside from the deployment bucket) is registered as the transfer bucket.

In addition, it supports two different file formats:

* `awsconfiguration.json` is the native format for the AWS Mobile SDK for Android and iOS
* `aws-exports.js` is the javascript format for AWS Amplify library for JavaScript, React, Angular, Ember, Vue and React Native.

## Conclusion

I hope this plug-in helps developers who have a significant investment in Serverless Framework to continue to use that investment — yet still get the benefits of the AWS Mobile SDK and AWS Amplify libraries. The plugin is far from perfect, but feel free to leave questions, bugs, and feature requests on [the GitHub repository](https://github.com/awslabs/aws-amplify-serverless-plugin).

{% include links.md %}
