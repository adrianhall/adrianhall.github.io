---
title: "What AWS service should you use to publish a web site?"
categories:
  - Cloud
tags:
  - "zCloud:AWS"
---

There are at least five different services you can use to publish a web site on AWS:

* Amazon S3 + Amazon Cloudfront
* AWS Amplify Console
* Amazon Lightsail
* AWS Elastic Beanstalk
* Do-it-yourself compute / storage / network stack

Which one of these do you choose? Well, “it depends” is a great answer to this question, but I hope that this article will demystify the strengths of each choice and give you a roadmap to make a choice.

This article is not designed for the person who knows a lot about AWS services. It glosses over a lot of details and those details may be very relevant to your specific use case. However, if you are lost in the myriad choices that AWS provides, hopefully this will provide some guidance on where to concentrate your research.

I have also deliberately not discussed pricing. The pricing models for each combination of services are different and inevitably will depend on how you expect the site to be used. You can and should do a comparison of the various options based on your specific use case. Pricing information is available for all options on [the AWS website](https://aws.amazon.com/pricing/).

## Amazon S3 + Amazon CloudFront

If your web site is just a collection of HTML, CSS, and JavaScript with no other dependencies, and you receive this from a marketing or design firm, you can just drag and drop the files onto your S3 bucket and get them deployed easily. This option may be for you. It’s simple to understand — you just publish your files to the S3 bucket and CloudFront picks them up and distributes them worldwide, also handling things like HTTPS for you.

I find that this is becoming less and less appealing, given the availability of AWS Amplify Console, but for those basic sites, this is probably the most cost effective way of getting a web site up and runningt.

**Best for**: Pre-packaged static sites provided by marketing organizations that are deployed via drag and drop.

## AWS Amplify Console

As the newcomer service, the [AWS Amplify Console] is an awesome continuous deployment (CD) platform for your web site. It has built in support for static site generators such as Gatsby, Jekyll, and Hugo, and support for JavaScript single page applications written in a variety of frameworks like React, Angular, or Vue. The AWS Amplify Console especially shines if you wish to deploy a SPA with a serverless backend built with the [AWS Amplify] CLI.

The service is very hands off — check your code into your source code repository (GitHub, CodeCommit, etc.) and a deployment will happen. Worldwide distribution happens via Amazon CloudFront so your app is responsive and planet scale.

**Best for**: Static site generators, JavaScript based single page applications.

## Amazon Lightsail

If your app relies on a web language backend (like Ruby or PHP) or you use a common web site platform (like WordPress or Magento), then you might want to choose [Amazon Lightsail]. Underneath, your Lightsail instance is comprised of virtual machines that run the appropriate software. You can run both Linux and Windows instances with Lightsail. It can be considered a crafted and opinionated set of virtual machines specifically designed around web applications. You get the ability to set up DNS, static IP addresses, load balancers, and connectivity to a VPC to access private resources like RDS hosted databases.

**Best for**: Common web stacks like LAMP, MEAN, and PHP, or common web applications like WordPress, MediaWiki, and Magento.

## AWS Elastic Beanstalk

On the surface, you will find a lot of similarities between Amazon Lightsail and [AWS Elastic Beanstalk] with EC2 instances. They both run exactly the same web application sets. They both run virtual machines. However, AWS Elastic Beanstalk can run within a VPC, which means you can run the web applications in an “internal only” configuration (with no connection to the Internet. For example, running MediaWiki as an internal-only employee information service). You also get a whole host of flexibility in terms of settings that you don’t get with Lightsail. One of the under-utilized features is blue-green deployments, which allows you to deploy and warm up a replacement web application while the existing web application continues to run. You can also configure the underlying OS that is running any way you need to, which opens up possibilities that aren’t available in the other options.

**Best for**: Your most challenging enterprise apps where access to the underlying OS is required.

## Do it yourself

Most of the time, when I can’t get away with a serverless backend and AWS Amplify console, I turn to a Docker container to provide the service. This docker container can be deployed via a CI/CD pipeline, scaled with Amazon ECS and managed as part of a complex stack. Doing things yourself means the maximum flexibility and control of the environment, but also the maximum amount of time spent managing the environment.

**Best for**: Your most challenging enterprise apps where you can use a variety of AWS services to augment your service offering.

## There are options

I’m sure there are other options available. AWS is a very flexible cloud with lots of options for everything you might want to run. I try to simplify what I need to manage by choosing the right tool for the application I am going to run. From basic static sites to the latest SPA applications with planetary scale backends, there is an option for each and every application you run.

{% include links.md %}