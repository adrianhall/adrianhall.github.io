---
title: "Bicep, loops, and defaults"
categories:
  - Azure
tags:
  - API Management
  - Bicep
---

I've been playing around a lot with [bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview) recently.  I like it because it is much more readable than ARM templates and lets me modularize my deployments easily.  Recently, I was writing a module for creating named values in Azure API Management.  Here is my `service.bicep`:

{% highlight terraform %}
@description('The name of the API Management instance to deploy this API to.')
param serviceName string = 'apim${uniqueString(resourceGroup().id)}'

@description('Location for all resources')
param location string = resourceGroup().location

module apimService './modules/api-management-service.bicep' = {
  name: serviceName
  params: {
    serviceName: serviceName
    location: location
    namedValues: [
      { key: 'petstoreUrl', value: 'https://petstore3.swagger.io/api/v3' }
      { key: 'validationKey', secret: true, value: 'my-validation-key' }
    ]
  }
}
{% endhighlight %}

Now, it doesn't matter why I need these to be the way they are - they could be anything.  The important thing to note here is that I only want the important things to be specified.  My first pass at creating the named values was to use this bicep snippet:

{% highlight terraform %}
resource namedValue 'Microsoft.ApiManagement/service/namedValues@2022-04-01-preview' = [for nv in namedValues: {
  name: nv.key
  parent: apimService
  properties: {
    displayName: nv.key
    secret: nv.secret
    value: nv.value
  }
}]
{% endhighlight %}

Well, that doesn't work.  The non-existence of a property in an object is an error, and will throw validation errors when you try to deploy the bicep template. Fortunately, there are two ways to get around this.  Firstly, the simple case:

{% highlight terraform %}
resource namedValue 'Microsoft.ApiManagement/service/namedValues@2022-04-01-preview' = [for nv in namedValues: {
  name: nv.key
  parent: apimService
  properties: {
    displayName: nv.key
    secret: contains(nv, 'secret') ? nv.secret : false
    value: nv.value
  }
}]
{% endhighlight %}

The ternary operator together with the `contains()` function allows you to ensure you never accidentally access an optional property.  However, this becomes untenable when you have lots of properties.  In this case, you will want to construct a new array based on the parameter:

{% highlight terraform %}
// Construct the named values with the default secret value.
var actualNV = [ for i in range(0, length(namedValues)): union({ secret: false }, namedValues[i]) ]

resource namedValue 'Microsoft.ApiManagement/service/namedValues@2022-04-01-preview' = [for nv in actualNV: {
  name: nv.key
  parent: apimService
  properties: {
    displayName: nv.key
    secret: nv.secret
    value: nv.value
  }
}]
{% endhighlight %}

The `union()` function applies properties from first to last, so later parameters to the `union()` function will overwrite properties that already exist.  

Let's take another example - [policy fragments](https://learn.microsoft.com/azure/api-management/policy-fragments).  I do a lot with [Azure Mobile Apps](https://github.com/azure/azure-mobile-apps).  That library demands that you include a specific header.  You can define a policy fragment to check the header like this:

{% highlight xml %}
<fragment>
    <check-header name="zumo-api-version" failed-check-httpcde="400" failed-check-error-message="Bad Request" ignore-case="true">
        <value>2.0.0</value>
        <value>3.0.0</value>
    </check-header>
</fragment>
{% endhighlight %}

Then you can include it wherever you need to in your policy documents with `<include-fragment/>`.  When you deploy a policy fragment, it needs a `fragment-id`, `description`, and `value`.  So, the following would normally be ok:

{% highlight terraform %}
resource policyFragment 'Microsoft.ApiManagement/service/policyFragments@2022-04-01-preview' = [ for pf in policyFragments: {
  name: pf.fragmentId
  parent: apimService
  properties: {
    description: pf.description
    value: pf.value
    format: 'rawxml'
  }
}]
{% endhighlight %}

However, again - what happens if `pf.description` is not set?  An undefined property is not just ignored - it is a validation error.  We could do what we did initially - use `contains()` inside a ternary operator.  However, we can also set a default value to a known value, but what value do we set if we want the description to match the fragmentId?  We can use `null` or a blank string (either should work) in this case:

{% highlight terraform %}
// Construct the policy fragments with the default value for description.
var actualPF = [ for i in range(0, length(policyFragments)): union({ description: '' }, namedValues[i]) ]

resource policyFragment 'Microsoft.ApiManagement/service/policyFragments@2022-04-01-preview' = [ for pf in actualPF: {
  name: pf.fragmentId
  parent: apimService
  properties: {
    description: pf.description == null || pf.description == '' ? pf.fragmentId : pf.description
    value: pf.value
    format: 'rawxml'
  }
}]
{% endhighlight %}

We can use the ternary operator to decide which value to use.  The ternary operator selects between two values depending on a condition (like if the description provided is blank).  In the var line, the description is guaranteed to be set and that is picked up later to set the deployed value of the description to the right thing.

I wish that bicep had the `?.` operator (to say "if the property doesn't exist, set to null") and the `??` null-coalescing operator - both of which are available in modern languages like C#.  This makes the provisioning of default values much easier.  But it doesn't and hence we have to find other ways to set defaults.
