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

There are other things that could happen in light of error handling, but that's the basics of it.  Let's look at some of the code.  First, the initialization:

{% highlight javascript %}
const cosmos = require('@azure/cosmos');
const EmailValidator = require('email-validator');
const uuid = require('uuid');

const sendgrid_connectionstring = process.env['SENDGRID_CONNECTIONSTRING'];
const cosmosdb_connectionstring = process.env['COSMOSDB_CONNECTIONSTRING'];

// Method to parse the connection strings
function parseConnectionString(connectionString) {
    const segments = connectionString.split(';');
    const result = {};
    segments.forEach((item) => {
        const [ key, value ] = item.split('=');
        result[key] = value;
    });
    return result;
}

// Parse the Cosmos database connection string
const cosmosSettings = parseConnectionString(cosmosdb_connectionstring);
const cosmosClient = new cosmos.CosmosClient({ 
    endpoint: cosmosSettings.Endpoint,
    key: cosmosSettings.Key
});

// Storage for the Cosmos databae and container client objects.
let database = undefined;
let container = undefined;
{% endhighlight %}

We bring in a few libraries (that I have used `npm install --save` to install into the `package.json` file).  Then we create a connection to the Cosmos account.  Finally, we have a couple of containers for the database and container within Cosmos.  This function may be run repeatedly and we don't want to try and create the database and container continually.  

Next is the actual function.  We start by creating the database and container if required, filling in the global variables if not.  Then we check the body of the request and do some validation:

{% highlight javascript %}
module.exports = async function (context, req) {
  // Determine if the database and container exist.
  if (!database) {
      const result = await cosmosClient.databases.createIfNotExists({ id: 'tailwind' });
      database = result.database;
  }
  if (!container) {
      const result = await database.containers.createIfNotExists({ id: 'profiles' });
      container = result.container;
  }

  const body = req.body || {};

  // Check that the body is complete.  It must have the following fields:
  //      email - a valid email address
  //      name - a full name, minimum 3 characters and max 64 characters
  if (!body.hasOwnProperty('email')) {
      context.res = { status: 400, body: 'No email field' };
      return;
  }
  if (!EmailValidator.validate(body.email)) {
      context.res = { status: 400, body: 'Email is invalid' };
      return;
  }

  if (!body.hasOwnProperty('name')) {
      context.res = { status: 400, body: 'No name field' };
      return;
  }
  if (body.name.length < 3 || body.name.length > 64) {
      context.res = { status: 400, body: 'Name is invalid' };
      return;
  }

  // Rest of the code
}
{% endhighlight %}

Next, we look for the user.  This is a straight Cosmos database query on the container:

{% highlight javascript %}
  // Find the record (if any) within the database container
  const querySpec = {
      query: "SELECT * FROM Profiles p WHERE p.email = @email",
      parameters: [
          { name: "@email", value: body.email }
      ]
  };
  const { resources: results } = await container.items.query(querySpec).fetchAll();
  // We should have 0 or 1 records - no more.
  if (results.length > 1) {
      context.res = { status: 500, body: 'Too many results for email address' };
      return;
  }
{% endhighlight %}

Now we have two situations.  Either the user has presented us with a registration code or they haven't.  Let's take the case when they don't present the registration code.  In this case, we want to generate one, create a record for the user, then tell the user what it is before returning:

{% highlight javascript %}
  // If there is no code, then the email record should not exist
  if (!body.hasOwnProperty('code')) {
      if (results.length != 0) {
          context.res = { status: 400, body: 'User already registered' };
          return;
      }

      // Add a record into the Cosmos container
      const registrationCode = '000000';      // This is the registration code
      const registrationRecord = {
          id: uuid.v4(),      // This is a new record
          state: 'PENDING',
          registrationCode,
          email: body.email,
          name: body.name,
          // Authentication code goes here
      };
      await container.items.upsert(registrationRecord);

      // Send the email to the user with the code
      context.log(`Registration Code = ${registrationCode}`);

      // Once done, return 201 Created
      context.res = { status: 201, body: 'Registration created - check your email' };
      return;
  }
{% endhighlight %}

I've obviously glossed over a bunch of problems here.  Firstly, there is an "authentication code goes here" marker.  I need to store the sub for the user authentication record so a user can't just register as one person and pretend to be another.  Also, I've skipped over the whole sending email piece - that's for another day.  Finally, I haven't even generated a registration code.  I've just hard-coded it.  This allows me to effectively test the function outside of the app and ensure the app logic is doing the right thing, but I definitely need updates before I go live.

In the other case, we need to check the registration code against the stored code and update the record if it is available.

{% highlight javascript %}
  // If there is a code, then there must be 1 record, and it must have state=PENDING and code=same
  if (results.length == 0) {
      context.res = { status: 400, body: 'No such user registered' };
      return;
  }
  const profile = results[0];
  // If the right code was entered...
  if (profile.state === 'PENDING' && profile.registrationCode === body.code) {
      // Update record to state = REGISTERED and remove code
      profile.state = 'REGISTERED';
      delete profile.registrationCode;

      // Update the record
      const item = container.item(profile.id, undefined);
      await item.replace(profile);

      // Once done, return 200 OK
      context.res = { status: 200, body: 'Registration complete - get a token' };
      return;
  }
  // If the wrong code was entered...
  if (profile.state === 'PENDING' && profile.code !== body.code) {
      context.res = { status: 401, body: 'Unauthorized - invalid code' };
      return;
  }
  // If the user is already registered...
  if (profile.state === 'REGISTERED') {
      // Just tell the user they are registered already
      context.res = { status: 200, body: 'Registration complete - get a token'};
      return;
  }

  // Anything else...
  context.res = { status: 500, body: 'Unable to complete registration request' };
{% endhighlight %}

This is fairly straight forward to read.  However, again, I'm glossing over the various security issues that are present here.  I don't check that the user who is sending me the code is the same user who requested the code, for example.  That's for later on.

## Deploy the code

I can deploy this function directly from the Visual Studio Code.  Firstly, click on the **Azure** icon in the left hand bar.  If you haven't already, sign in.

![](/assets/images/2019-09-05-image1.png)

Right-click on the `twprodfunctions` function app and select **Deploy to Function App...**.  The process takes a couple of minutes, but eventually will complete.  From here, you can also view streaming logs.

Once you have deployed, bring up [Postman](https://getpostman.com) and do a POST to the `/api/register` endpoint with the following body:

{% highlight json %}
{
  "name": "Your Name",
  "email": "youremail@youremail.com"
}
{% endhighlight %}

You can actually cut and paste this in since we aren't doing validation yet.  The first time through, you will see a short delay.  This is the "cold start".  Eventually, the process will complete with a `201 Created`.  

![](/assets/images/2019-09-05-image2.png)

You can go to the Cosmos database within the Azure portal and check to see if the item is created:

{% highlight json %}
{
  "id": "20050d84-78e2-4b8f-a1df-0c1e63cb0940",
  "state": "PENDING",
  "registrationCode": "000000",
  "email": "your@email.com",
  "name": "Your name",
  "_rid": "MUxbAPH4TlwEAAAAAAAAAA==",
  "_self": "dbs/MUxbAA==/colls/MUxbAPH4Tlw=/docs/MUxbAPH4TlwEAAAAAAAAAA==/",
  "_etag": "\"77009997-0000-0700-0000-5d7050e20000\"",
  "_attachments": "attachments/",
  "_ts": 1567641826
}
{% endhighlight %}

You can now alter the JSON document in Postman to the following:

{% highlight json %}
{
  "name": "Your Name",
  "email": "youremail@youremail.com",
  "code": "000000"
}
{% endhighlight %}

POST that and see the `200 OK` result.  Also, check the result of the database update:

{% highlight json %}
{
  "id": "20050d84-78e2-4b8f-a1df-0c1e63cb0940",
  "state": "REGISTERED",
  "email": "your@email.com",
  "name": "Your name",
  "_rid": "MUxbAPH4TlwEAAAAAAAAAA==",
  "_self": "dbs/MUxbAA==/colls/MUxbAPH4Tlw=/docs/MUxbAPH4TlwEAAAAAAAAAA==/",
  "_etag": "\"7700b09a-0000-0700-0000-5d7051850000\"",
  "_attachments": "attachments/",
  "_ts": 1567641989
}
{% endhighlight %}

You can also take a look at what happens when the user is not registered but you submit a code, and what happens when the code is wrong if you like.



