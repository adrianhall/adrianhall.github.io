---
title: "Building a service in the Cloud"
categories:
  - Mobile
tags:
  - Serverless
---

I’ve been thinking about providing services in the cloud for a few years now.    A common question I see time and time again is this:  “What pieces of the cloud do I need to build my solution?”  The answer is always “it depends.”  This post is about those options.

A service – whether it be a mobile backend, web server or public API – generally has different kinds of content.  A web site, for example, will have images, stylesheets and other static content.  However, that same web site may also communicate with an API that generates dynamic content.

If you are a hobbyist or just starting out in mobile app development, it is very tempting to use one of the MBaaS (Mobile Backend as a Service) solutions out there – Google Firebase or Azure App Service Easy Tables, for example.  However, each of these has a natural brick wall.  You can’t do everything in an MBaaS.  You are trading flexibility for simplicity.  Things that should be in the server, like business logic or security, end up in the client.  When you hit that wall in terms of complexity, you will need to migrate your data and users to something else.  MBaaS solutions provide a large amount of free services and they may be perfect for your particular solution, but they are not for every solution.  You need to project your needs out beyond the immediate and look at 2-3 years down the road.  What will be the pain associated with migrating your application data and users to another provider because the MBaaS ran out of flexibility.  Security, in particular, belongs in the service, not the client.

Here are some of my general rules in considering what cloud services to use:

1. Get as close to the source as you can.
2. Abstract away from the virtual machine as much as you can.
3. Don’t pay for more than you use.
4. Don’t ship secrets in your app.

Simple rules.  Let’s take a look at these:

## Get as close to the source as you can

Inevitably, you will want to use a database of some form for dynamic content.  SQL databases (such as MySQL, PostgreSQL, Oracle or SQL Server) are not built with the web in mind.  There is no REST interface to them.  They all require a stateful connection to the database.  The web is inherently stateless.  That means you are going to have to run an intermediary.  MySQL has the [MySQL HTTP Plugin](http://blog.ulf-wendel.de/2014/mysql-5-7-http-plugin-mysql/) in its latest release that it suggests running on a separate server.  PostgreSQL has a number of plugins and options for providing a REST API with the database.  Both of these assume that you are running the database service for yourself and it is not provided as a service.  Inevitably, you will end up writing a REST API service and running it in a web hosting facility.

Compare that typical NoSQL databases. AWS DynamoDb, Azure DocumentDb, and Google Firebase Realtime Database all allow communication over a web protocol in a stateless manner.   This is ideal for web and mobile development.  Your mobile or web application can go direct to where you store the data – the database.

Similarly, you are storing those static resources in some sort of storage.  You should ensure that storage has a web interface already (most cloud storage do) and that you can provide public access tokens to the storage (which is more tricky).  AWS S3, for example, allows you to [upload a bunch of static resources and then just make them public](http://docs.aws.amazon.com/AmazonS3/latest/dev/HostingWebsiteOnS3Setup.html).  You do not need to go through an intermediary to get access to them.

When there are less intermediaries, there is less to go wrong.

## Abstract away from the virtual machine as much as you can

You’ve figured out that you need some sort of compute resource in the cloud.  Perhaps you are accessing a SQL database, or providing tokens to your app that requires configuration in the cloud.  Whatever the reason, there are options:

* **Serverless** (AWS Lambda, Azure Functions, Google Cloud Functions) is best for APIs that are stateless.  For example, if you are providing your mobile app a security token based for the NoSQL database, this is best done in the serverless world.  There is no need to worry about the machine at all with Serverless.  Everything is abstracted away.  However, the price you pay for this is that your code cannot reliably maintain state (unless you can persist that state in storage).  That means it is not a good place to be doing REST to SQL.
* **Containers** (AWS ECS, Azure Container Service, Google Container Engine) or **Web PaaS** (Azure App Service, Google App Engine) are best for the majority of your dynamic APIs that need to maintain state.  You will be paying for the entire time they are running; not just when you get a request like with Serverless.  However, you will be able to maintain state here.  You will inevitably need to specify the size of the virtual machine (by CPU/memory) that you are running on but you don’t need to manage that virtual machine (security patching, monitoring, etc.)  That is done for you by the platform.
* **Virtual Machines** (AWS EC2, Azure Virtual Machines, Google Compute Engine) is the choice of last resort.  Use this only if you need specific capabilities of the platform of choice.   An example of this is that you need to use a specific library that is only available on that platform.  For example, Microsoft provides Authenticode for handling code signing.  This only works on Windows (surprise!)  You would need to use a virtual machine if you were writing an Authenticode code-signing service.

## Don't pay for more than you use

Thus far, I’ve suggested storage (which is generally charged by the kB), serverless (which is generally charged by the CPU-second), containers, web PaaS, and virtual machines (which all tend to be charged by the size of the virtual machine).  Storage and Serverless are the cheapest (in small quantities) whereas persistent machines are expensive.  If you put static content in your container, you are going to be paying more for serving that static content.

In small apps, where all the APIs and static content can reasonably live inside the smallest container, putting everything in the same container is reasonable.  As your app grows, you should plan to split it apart to get better cost efficiencies from the cloud.  Splitting the component services apart allows you to reduce the size of or eliminate the container/virtual machine.  That means less running costs over time.

## Don't ship secrets in your app

Anything you ship in your app is public.  Don’t ship secrets.  That includes API Keys.  Instead, provide an API that distributes the URL and API keys of your various components to your app.  Your service library might cache this information, but you can allow it to change over time.

How will you stop other developers using your APIs without permission?  Well, there is a hard lesson to learn here and no way for me to gently let you down.  You need to request that your users authenticate.  There is no mechanism available that provides a fool-proof way of authenticating an app to your service.  You authenticate users.

There is, of course, a non-fool-proof way:

* Take a checksum of your app (or something else you can measure before shipping and that your app can measure for itself, but is not embedded in your app)
Create a JSON Web Token with a request signed by the checksum of your app and submit that to your authenticator.
* Your authenticator will check the JSON Web Token against the known app values and ensure the contents are valid, then produce a new token signed for the service.
* Your other service APIs will look for the new token as authentication.

This is middle-ground.  It still is not secure (and I’m sure someone with a crypto or security background will point that fact out), but it’s better than deluding yourself into thinking an API key is secure.  API keys are for routing, not security.

## How I structure cloud backends

The majority of my cloud backends are written in a modular manner these days.

* Static Content starts life in the container if I am using one or storage if I am not.  As it grows, I’ll move it to storage or CDN.
* A configuration API runs in Serverless compute unless I am using a container, in which case the configuration API runs on the container.
* The mobile app communicates directly with a NoSQL database.
* If the mobile app needs to communicate with a SQL database, the API is wrapped (typically as a Node/Express or ASP.NET Core service) within a Container.

I only distribute the URL of the configuration API within my mobile app.  Everything else is configured after a call to the configuration API. I design the configuration API to be fast – no database lookups allowed.

Of course, that’s not the last word.  Cost and security are the main concerns for cloud backend development.  However, organizations have their own priorities.  It may be that the organization requires that the cloud backend all be stored within the same source code repository, be capable of being deployed as  a single container for local development purposes, or be co-located in a single security zone.   These requirements tend to reduce flexibility choices in the cloud and increase costs at larger scales as they force architectural decisions in conflict with the principals I’ve talked about in this post.
