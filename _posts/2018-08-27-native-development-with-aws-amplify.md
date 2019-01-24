---
title: "Native Development with AWS Amplify"
categories:
  - AWS
  - Mobile
tags:
  - "AWS Amplify"
---

Up until this point, the experience for developing native iOS and Android apps with AWS has been a bit fragmented. There was AWS Mobile Hub; a web console that makes it easy to provision serverless cloud resources and get a configuration file back that describe the endpoints your app needs to communicate with. But because it is a web-based console, it requires the developer to switch back and forth between their local development machine and the browser. There is also the awsmobile CLI, but that is targeted at JavaScript developers. It provisioned the same AWS Mobile Hub resources, but generates a JavaScript based configuration file — not the appropriate format for iOS and Android developers.

If you wanted to go further afield, you could use one of [AWS CloudFormation], [Serverless Framework](https://serverless.com/), or [Terraform](https://www.terraform.io/) to provision the resources. These have the advantage that your infrastructure deployment instructions can be checked into the same repository as your app, but they lack the ability to generate the appropriate configuration file necessary to describe the infrastructure to your app.

Last week, AWS released the [AWS Amplify] CLI, and it supports native developers out of the box. That means you can create a completely serverless backend suitable for web and mobile applications and get the appropriate configuration file necessary to communicate with that backend, all from the comfort of a terminal window, without going to the web console. It also means that the configuration file is automatically kept in sync with the changes to your backend.

Let’s start by installing the CLI. You need Node installed on your system, then it’s just a global Node application:

{% highlight bash %}
npm install -g @aws-amplify/cli
{% endhighlight %}

You also need to tell it how to connect to your AWS account:

{% highlight bash %}
amplify configure
{% endhighlight %}

If you have the AWS CLI installed already, then you can use a pre-existing profile, which configures the access key, secret key and region information for deploying releases. If not, the command will walk you through the one-time configuration of a user that you can use. Check out this handy video if you need a walkthrough:

<iframe id="player" frameborder="1" allowfullscreen="1" allow="autoplay; encrypted-media" title="YouTube video player" width="640" height="360" src="https://www.youtube.com/embed/fWbM5DLh25U?wmode=opaque&enablejsapi=1&origin=https%3A%2F%2Fcdn.embedly.com&widgetid=1"></iframe>

## Add AWS Amplify to your app

The same process is used irrespective of whether your app is Android or iOS. Open up a terminal and change your directory to your app root. This should be the top level that is checked into your source code repository. Run the following command:

{% highlight bash %}
amplify init
{% endhighlight %}

There are a bunch of questions that need to be answered. The important one is to ensure the appropriate project type is selected. The Amplify CLI tries to detect whether your app is an Android or iOS app. If it detects the source code format, it will preselect the right thing. If not, you will need to create it. Now, let’s add analytics to our project.

Why analytics? My belief is that every single app needs to understand how it is being used. Analytics is a baseline requirement that the cloud can support. It’s also free for most users.

This command creates an `amplify` directory to store the infrastructure information. You should definitely check the `amplify/backend` directory into source code control.

{% highlight bash %}
amplify analytics add
{% endhighlight %}

Enable guest usage — you can accomplish that by pressing enter when asked a question. This command creates a CloudFormation template in the `amplify/backend` directory which has the Amazon Pinpoint definition, plus IAM rules for allowing your mobile app to write to the incoming pipeline. It does not create the resources yet. To do that:

{% highlight bash %}
amplify push
{% endhighlight %}

After a confirmation (this is what we are going to do!), it deploys the new service. Once deployment is complete, you will have some extra stuff:

* An `amplify/#current-cloud-backend` directory — this contains the deployed version of the CloudFormation templates.
* An `awsconfiguration.json` file — this is the configuration file that you need to integrate with your app.

The `awsconfiguration.json` file is placed in the “right location” for your app — in the `res/raw` directory for an Android app, or the root directory for an iOS app. If you are developing in Xcode, you need to add the `awsconfiguration.json` file to your Xcode project (just once) using **File** > **Add File…** Make sure to uncheck **Copy to…** as this allows the project to sync with any future configuration changes.

## It's just CloudFormation

One of the beautiful things here is that the deployment script is pure [AWS CloudFormation], which is the standard way of automatically deploying resources on the AWS cloud. The `amplify` CLI generates a nested stack. The root stack contains some IAM roles, and then there is a stack per category — one for auth, one for analytics, one for storage, etc.

Right now, the AWS Amplify CLI supports the following capabilities:

* Analytics (with [Amazon Pinpoint])
* Authentication (with [Amazon Cognito])
* REST API (with [Amazon API Gateway] and [AWS Lambda])
* GraphQL API (with [AWS AppSync])
* Push Notifications (with [Amazon Pinpoint])
* Storage (with [Amazon S3])
* Functions (with [AWS Lambda])

You can also deploy static resources via [Amazon CloudFront] and [Amazon S3] with a hosting module. Each of these has been thought through so that you are only asked for settings relevant to a mobile developer, so it’s likely you will never have to worry about the CloudFormation templates. If you want to change anything, you can edit the CloudFormation templates pre-deployment or adjust the resources post-deployment in the web console.

Some commands produce additional files that are necessary to integrate the service with your app. For example, the REST API generates source code to allow you to communicate easily with the API gateway, and the GraphQL API generates the appropriate schema file for handling that functionality. This ensures that you never have to touch the AWS web console in your day-to-day development tasks.

## Conclusion

Of course, there are times you want or need to use the web console. For example, you might want to view analytics (`amplify analytics console` gets you there) or check to see if a database insert actually stored the right data.

The [AWS Amplify] CLI makes it easier to work with your mobile backend from within your development workflow. Now, it doesn’t matter if you are an iOS or Android native developer, React Native, or web developer. You can use the same tools and get the same effect — an awesome scalable and secure backend for your mobile app.

{% include links.md %}
