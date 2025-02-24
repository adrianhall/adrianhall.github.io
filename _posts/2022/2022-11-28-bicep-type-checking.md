---
title: "Type-checking Bicep arrays and objects"
categories:
  - Azure
tags:
  - Bicep
---

As you may have guessed by now, I'm delving heavily into the world of [Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) right now, mostly in order to describe the infrastructure for my personal projects in a readable way.  JSON and YAML (used by ARM) is most definitely not readable for the average consumer.  Part of that work was learning about [bicep modules](https://learn.microsoft.com/azure/azure-resource-manager/bicep/modules), which I love for modularizing my code.  However, there is one distinctive problem with this.

Let's take an example to show off the problem.  I have a bicep fragment that looks like this:

{% highlight text %}
@description('List of named values that this API depends on.')
param namedValues array = []

resource apimService 'Microsoft.ApiManagement/service@2022-04-01-preview' existing = {
  name: serviceName
}

resource apimNamedValue 'Microsoft.ApiManagement/service/namedValues@2022-04-01-preview' = [for nv in namedValues: {
  name: nv.key
  parent: apimService
  properties: {
    displayName: nv.key
    secret: contains(nv, 'secret') ? nv.secret : false
    value: nv.value
  }
}]
{% endhighlight %}

This is actually from my bicep module for deploying an OpenAPI specification to Azure API Management.  You can see from reading the snippet that each element in the array of named values needs a key and value, and optionally takes a secret.

How does the consumer of this module know that?  How can Visual Studio Code provide Intellisense type help to the consumer to facilitate this knowledge?

This problem has been solved as a preview feature in the latest release of bicep.

## Upgrading bicep

First off, let's upgrade bicep:

{% highlight bash %}
$ az bicep upgrade
$ az bicep version
Bicep CLI version 0.12.40 (41892bd0fb)
{% endhighlight %}

Now, let's enable the preview feature.  You need to [create a `bicepconfig.json` file](https://learn.microsoft.com/azure/azure-resource-manager/bicep/bicep-config) in the root of the project.  My main bicep file is always called `main.bicep` and I put the `bicepconfig.json` file alongside that file.  Here are the contents:

{% highlight json %}
{
    "experimentalFeaturesEnabled": {
        "userDefinedTypes": true
    }
}
{% endhighlight %}

If you have the bicep extension installed for Visual Studio Code, then Intellisense will be enabled for this file to guide you.

## Creating an object type

Both objects and arrays are affected by this, any my type checking is for an array of objects. I first need to define the shape of the object:

{% highlight text %}
type NamedValue = {
  @minLength(1)
  @maxLength(63)
  key: string

  @minLength(1)
  value: string
  
  secret?: bool
}
{% endhighlight %}

I love the fact that I can use the same validation on the properties of the object as I do on parameters.  Also, note the syntax for the optional `secret` property.  Make sure you define the types at the top of the file, especially if you are going to use them as types in your parameters.

## Creating an array type

Next, I need to update my parameter definition:

{% highlight text %}
@description('List of named values that this API depends on.')
param namedValues NamedValue[] = []
{% endhighlight %}

Instead of the keyword `array`, you use the monikor `Type[]` to specify an array of a specific type.  If the array is a an array of standard types (e.g. strings), you can use those types as well:

{% highlight text %}
@description('A list of strings')
param alternatePaths string[]
{% endhighlight %}

## Putting it together

Let's look at the completed work:

{% highlight text %}
type NamedValue = {
  @minLength(1)
  @maxLength(63)
  key: string

  @minLength(1)
  value: string
  
  secret?: bool
}

@description('The name of the API Management service to deploy the API to.')
@minLength(1)
param serviceName string

@description('List of named values that this API depends on.')
param namedValues NamedValue[] = []

resource apimService 'Microsoft.ApiManagement/service@2022-04-01-preview' existing = {
  name: serviceName
}

resource apimNamedValue 'Microsoft.ApiManagement/service/namedValues@2022-04-01-preview' = [for nv in namedValues: {
  name: nv.key
  parent: apimService
  properties: {
    displayName: nv.key
    secret: contains(nv, 'secret') ? nv.secret : false
    value: nv.value
  }
}]
{% endhighlight %}

This module will allow you to define named values as a list:

{% highlight text %}
module apim_named_values './modules/apim-named-values.bicep' = {
  name: 'apim_named_values'
  params: {
    serviceName: apimServiceName
    namedValues: [
      { key: 'nv1', value: 'value1' }
      { key: 'nv2', secret: true, value: 'value2' }
    ]
  }
}
{% endhighlight %}

I am using this mechanism to deploy both named values and policy fragments for my Azure API Management instances.  Visual Studio Code tells you in the code editor what is required:

![]({{ site.baseurl }}/assets/images/2022/11-28/image1.png)

Hopefully, this feature comes out of preview soon!
