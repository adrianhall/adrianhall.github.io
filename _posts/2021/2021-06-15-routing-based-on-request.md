---
title: "Service Routing for Azure Mobile Apps with API Management"
categories:
  - Azure
tags:
  - Azure API Management
  - Azure Mobile Apps
---

Todays topic is "how do I upgrade the service backend to support the new ASP.NET Core service without affecting my current customers?"  Let's assume, for a moment, that I have already [created an API Management service]({% post_url 2021/2021-06-08-using-apim-with-zumo %}), and [added caching]({% post_url 2021/2021-06-11-adding-policies-to-apim %}) to my API.  Finally, I've adjusted the client application so everything is routing through the API Management service instead of direct to the backend.

Azure Mobile Apps uses a header to indicate the version of the protocol that is in use.  This head (`ZUMO-API-VERSION`) is set to `2.0.0` if you are using the ASP.NET Framework version of the server.  This version uses OData v3 to query the database.  The new ASP.NET Core version of the server uses OData v4 to query the database, so we upgraded the `ZUMO-API-VERSION` to 3.0.0 to indicate that this is a breaking change.  You can use the same database, but the service is completely different.

## Routing via Policy

When a request comes into API Management, I want to route the request to one of two services.  Unsurprisingly, this can be done with a policy.  Let's take a look:

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Find and select your API Management resource.
1. In the sidebar, select **APIs**, then select your API.
1. Select your API, then select **All operations**.
1. Open the policy editor for your API.

   ![Open the policy editor]({{ site.baseurl }}/assets/images/2021/2021-06-15-image1.png)

I've already got a `<check-header/>` policy, and the new policy will appear directly below that.  This time, I'm going to add a conditional policy statement that tests the value of the header:

{% highlight xml %}
<inbound>
    <base />
    <check-header name="ZUMO-API-VERSION" failed-check-httpcode="400" failed-check-error-message="Invalid ZUMO-API-VERSION Header" ignore-case="true">
        <value>2.0.0</value>
        <value>3.0.0</value>
    </check-header>
    <choose>
        <when condition="@(context.Request.Headers.GetValueOrDefault("ZUMO-API-VERSION","") == "2.0.0")">
            <set-backend-service base-url="https://zumo-net46.azurewebsites.net/tables/movies/" />
        </when>
        <when condition="@(context.Request.Headers.GetValueOrDefault("ZUMO-API-VERSION","") == "3.0.0")">
            <set-backend-service base-url="https://zumo-netcore.azurewebsites.net/tables/movies/" />
        </when>
    </choose>
</inbound>
{% endhighlight %}

Here, the `<check-header/>` makes sure we have a valid value, then that value is tested and the backend service is set accordingly.  The per-operation policies do not set the backend service.  If you had set a policy on an operation that set the backend, you would need to remove it.

Once you have saved the policy, it should start working within a minute.  You can test it by sending requests using the **Test** tab, as we have in previous articles.  Just set the `ZUMO-API-VERSION` and send a query.  The query structure for the two protocol versions is significantly different.

## Revisions and Versions

When you introduce a new functionality change in your backend or you change the policies attached to an API, there is a possibility (or even a probability) that something will break.  You don't want to break your users, so you want to ensure that the API exposed to the users has been fully tested.  To support this, Azure API Management has revisions and versions.  Revisions are for non-breaking changes, and Versions are for breaking changes.

Let's take versions first.  Versions are selected based on a change in the Uri path, a query parameter, or a header.  I could have implemented the version check I implemented above as a Version - two distinct versions, each one handling one of the version strings.  However, I prefer to keep the protocol check separate from a version check.

Let's say I wanted to introduce a breaking change into the `/tables/movies` API.  I'll call this the "2021-06-15" version of the API.  I want the version to be selected when I include a query string called `api-version` on my query.

1. Select the **APIs** section of your API Management resource.
1. Press the triple-dots next to the `Movies` API.
1. Select **Add version**.
1. Fill in the form:
    * Version identifier: `2021-06-15`
    * Versioning scheme: `Query string`
    * Version query parameter: `api-version`
    * Full API version name: `movies-2021-06-15`
1. Press **Create**.

You will see that you now have an "Original" and a "2021-06-15" entry under the Movies API:

![Two versions listed]({{ site.baseurl }}/assets/images/2021/2021-06-15-image2.png)

You can select between the two versions - they are completely independent. The Original is the default (it doesn't require a version tag) and the new one requires a version tag.  If you look on the **Settings** tab for each API, you will see both have a Version field now.

So, how could I have set this up so that the routing is different?  I have two versions.  I could set the version selector as the header `ZUMO-API-VERSION`.  Go into the original and set the Version identifier as `2.0.0` and the Web service URL to be my ASP.NET Framework service.  Repeat for the new version, but with the version identifier as `3.0.0` and the Web service URL as my ASP.NET Core service.  Hey presto, I've got two running versions.

Why would I do this?  Well, if I have diagnostic logging turned on punting the logs to Application Insights, I can monitor how many users are using the old server vs the new server relatively easily.  I can turn off the old server (and remove the version specification) once I know there is no more usage.

It's important to note that these two APIs are completely separate.  If I update one version with a changed policy, it doesn't get applied to other versions.

## Revisions for minor changes

So, what about small changes?  I like to create a **revision** every time I change a policy.  When you use revisions, the API is assumed to act the same - there is one "active" revision and all others must be manually selected.  Let's add a revision to one of the versions:

1. Select the version you want to edit, press the triple dots, and select **Add revision**.
1. Create a revision description (think of this as a change log), then press **Create**.

That was easy.  You can now make any changes to the API you want to make.  It won't affect the existing users because you are editing a new revision.  You can see this in the name - it will have `;rev=2` on the end.  When you go to test the API, you will see a Request URL like this:

* `https://aspnetcore-zumo.azure-api.net/tables/movies;rev=2/?api-version=2021-06-15`

This has both the version and a revision in it.  The backend does not see the revision or the version.  It just sees the original Uri that it is expecting.

Select the **Revisions** tab.  You will see two revisions:

![Two revisions listed]({{ site.baseurl }}/assets/images/2021/2021-06-15-image3.png)

Note that there is a "current" revision.  Every other revision is "not current".  The current revision is the one that is being used if a revision is not specified in the URL.  You can mark the new revision as "current" through the triple-dots menu next to the revision.

## Wrap up

So, there you have it - three different ways to control versioning - as a policy, as a version, and as a revision.  so, which one would I use?  Well, it depends.

Controlling versions via policy is good when the differences between versions are small and self-contained - like my case of handling different versions with different backends.  If I were doing more complex situations, then I would definitely switch over to versions.  Versions allow me to have completely independent policies for each protocol version, which can be a good thing, but is more to maintain.

I use revisions to introduce new incremental changes to my policies irrespective of whether I am using versions or policy to control the version.  It's a good practice to test your APIs before introducing them to the users.  My non-play apps all have a set of unit tests against the APIs.  I run the unit tests against the new revision prior to making the new revision current.  That way I can ensure I am not affecting my users.
