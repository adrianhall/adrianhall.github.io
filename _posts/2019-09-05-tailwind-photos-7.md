---
title: "Tailwind Photos: Registration (The Function)"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - Kotlin
  - "Azure Functions"
  - "Azure CosmosDB"
---

In [the last article I wrote]({% post_url 2019-09-03-tailwind-photos-6 %}), I introduced the architecture for my backend and created the resources that I am going to use within Azure.  However, I didn't write any code (except for the deployment script).  Today, I'm writing the initial code for the `/api/register` endpoint.  This is called when the user needs to register a new account.  It's called in two ways:

* `POST /api/register` initiates the registration process.  The response is:
  * `401 Unauthorized` if the social media auth token is invalid or not present.
  * `409 Conflict` if the email address already exists.
  * `201 Created` if the record was created successfully.
  * `503 Service Unavailable` if the database is not available.

The body may or may not contain a code.  The first time through, the code is not present.  In this case, a record will be generated within the database with a `state=PENDING` and a `code=XXXXXX` where the code is randomly generated.  The next time through, the user will submit a code.  If the code matches, then we update the `state=REGISTERED` and remove the code entry.

Let's take a look at how to create the function app.  I am doing this in [Visual Studio Code](https://code.visualstudio.com/) and have the following extensions enabled:

* [Azure Account](https://marketplace.visualstudio.com/items?itemName=ms-vscode.azure-account)
* [Azure Functions](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azurefunctions)

I've got a bunch of other extensions installed as well, but these are the ones you need.  I'm writing the code in JavaScript.

## Build a function app

Open up Visual Studio Code with the `tailwind-photos-backend` repository.  Then:

* **View** > **Command palette...**
* **Azure Functions: Create new project...**
* Select the current folder.
* **JavaScript**.
* **Azure Functions v2 (.NET Standard)**.
* **HTTP trigger**.
* Enter `register` for the function name.
* **Anonymous**.

At this point, a whole host of files will be scaffolded out for you, including a `register` folder that contains the actual function definition.  The function definition is two files:

* `register/function.json` configures the runtime with authentication level, input bindings, and output bindings.
* `register/index.js` is the actual code for the function.

The trigger is the input binding - we selected a HTTP trigger.  We're also defining a response.  Since the `/api/register` is a POST only API, let's adjust the `function.json` accordingly:

{% highlight json %}
{
  "bindings": [
    {
      "authLevel": "anonymous",
      "type": "httpTrigger",
      "direction": "in",
      "name": "req",
      "methods": [
        "post"
      ]
    },
    {
      "type": "http",
      "direction": "out",
      "name": "res"
    }
  ]
}
{% endhighlight %}

We using anonymous authentication in this case because we have not configured anything yet.  We'll deal with authentication processing later on. Now, let's take a look at the code for the function:

{% highlight js %}
module.exports = async function (context, req) {
    context.log('JavaScript HTTP trigger function processed a request.');

    if (req.query.name || (req.body && req.body.name)) {
        context.res = {
            // status: 200, /* Defaults to 200 */
            body: "Hello " + (req.query.name || req.body.name)
        };
    }
    else {
        context.res = {
            status: 400,
            body: "Please pass a name on the query string or in the request body"
        };
    }
};
{% endhighlight %}

Each function you expose is an async function.  For parameters, it takes a context (which includes a log method and whatever output bindings you need) and the input bindings (in this case, the request object).  You do whatever you need to do, setting the output binding object within the context.  Once complete, you return.

You can do per-function initialization outside of the function.  For instance, I need to read in the connection string of the Cosmos database from the environment, so I'm going to do that once rather than on every function invocation.  I will also create the Cosmos client object outside of the function so that every function invocation uses the same resources.

In the root directory is a `package.json` file.  You can use `npm` or `yarn` to install dependencies.  The underlying runtime is Node 10.14.1 (it was specified during the deployment), so anything that is normally available in that version of Node is also available here.

## The register function

Now that we've got the basics out the way, what does the `/api/register` function look like.  I'm going to assume that the user is authenticated (and you will see why later on - I promise!).  Here is what I have to do:

1. Load in the configuration from the app settings (which present as environment variables).
2. Create a connection to my Cosmos DB.
3. Create the required Cosmos databases and containers, if required.

Then, when a connection comes in:

1. Look up the user within the Cosmos database.
2. If the body contains a `code` entry:
    * Check to see if the `state=PENDING` and code matches.  If not, return `401` status.
    * If the code matches, then return the `/api/token` response.
3. If the body does not contain a `code` entry:
    * Create a new record in Cosmos, with a new Code.
    * Send email to the user with the code.
    * Return a `201 Created` status.

There are other things that could happen in light of error handling, but that's the basics of it.  Let's look at the code:



