---
title: "Tailwind Photos: Registration (The Azure Function)"
categories:
  - Android
  - "Tailwind Photos"
tags:
  - JavaScript
  - Kotlin
  - "Azure Functions"
---

A quick recap - we've got three identity providers integrated into our app, set up an Azure Functions App in our backend using ARM, and we've set up authentication on that function app.  We've also swapped our identity provider authentication token for an Azure App Service authentication token so we can use it on our backend.  Now it's time to consider the actual Azure Function for registration.

You want to store profile information in your backend.  In my case, it allows me to communicate with the user (since I store their email address) and it allows other people to find my users (since they will know the email address).  If I don't have a profile service, then users won't be able to see which photos are theirs if they log in with multiple providers.

The actual Azure Function is made up of two files.  The easiest way to make one is using the [Azure Functions Visual Studio Code plugin](https://code.visualstudio.com/tutorials/functions-extension/getting-started).  Azure Functions have triggers (things that cause them to run) and bindings (where they get their inputs and where they send their outputs).  For the registration API, we're going to trigger on a HTTP POST and the HTTP request will be the input binding and the HTTP response will be the output binding.  If you follow the tutorial linked above, you will get the right thing.

Let's take a quick look at the first file - the `function.json` file.  This defines the triggers and bindings.  It also determines the authentication level.  There are three auth levels - anonymous (no API key required), function (a function level API key required), or system (a function app level API key required).  I'm using anonymous auth level here:

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

Basically, my function is going to operate as a HTTP API - the input binding will be called `req` and the output binding will be called `res`.  It will be triggered by a HTTP POST.  Since it's in a directory called `register`, it will be called by doing a `POST /api/register`.

Let's turn our attention to the code.  The basic code is simple:

{% highlight javascript %}
// Initialization code

module.exports = async function (context, req) {
  // Function code
}
{% endhighlight %}

First thing to note - functions are async!  There is a synchronous version, but you won't need it for most purposes.  The `req` is the variable that is passed in from the input binding.  It must be named the same thing as the name within the `function.json` file.  The context contains a bunch of things, including the `res` variable that is the output binding.  It's on the context because there might be multiple of them.  Aside from that, the other thing to be aware of on the context is `context.log()`, which sends data to the logs (App Insights or the streaming logs).

## Step 1: Validate input

The request has the headers, query parameters, and body on it.  The body is decoded if it is JSON.  My API is going to take the email address and name in the body.  Most of the time, this will be provided by the IdP.  However, some IdPs (most notably, Sign in by Apple) are more privacy focused, and the IdP (or user) may not have given you permission to one of them.  I want the API to handle those cases as well.  Let's start exploring the code for this:

{% highlight javascript %}
module.exports = async function (context, req) {
  // Dump context information
  context.log(`env: ${JSON.stringify(process.env,null,2)}`);
  context.log(`hdr: ${JSON.stringify(req.headers, null, 2)}`);

  // The input is valid if all the following are true:
  //  1) We have an email and it's valid
  //  2) We have a name and it is more than 3 characters
  const input = {};
  context.log(`1. Checking email field`);
  if (req.body && req.body.email && isEmail(req.body.email)) {
      context.log(`1. Email field is valid - adding to input`);
      input.email = req.body.email.toLowerCase();
  } else {
      context.log(`1. Email field is invalid or not present - returning 406`);
      return handleError(context, 406, "Email is invalid");
  }

  context.log(`2. Checking name field`);
  if (req.body && req.body.name && req.body.name.length > 3) {
      context.log(`2. Name field is valid - adding to input`);
      input.name = titleCase(req.body.name);
  } else {
      context.log(`2. Name field is invalid or not present - returning 406`);
      return handleError(context, 406, "Name is invalid");
  }
  // In these cases, the 406 will trigger a "registration page"

  // REST OF CODE
}
{% endhighlight %}

You should put any validation you need here.  I've got a couple of "helper" methods in use here.  Once of them determines if the email address is valid, and the other converts a string to title case (i.e. the first character is upper-case).  In Functions work, I try to code the function rather than use a module.  Modules take time to load and that results in additional time during a cold-boot of the Function, so I try to avoid it.

The next step is to validate the authentication token.  This is where you need some [intimate knowledge of Azure App Service Authentication](https://docs.microsoft.com/en-us/azure/app-service/app-service-authentication-how-to).  There are several headers that App Service Authentication adds to your request:

* `x-ms-client-principal` is a base-64 encoded JSON blob with the contents of the id_token.
* `x-zumo-auth` is a JWT that is sent from the client and contains information to identify the user.  It's basically the same information as `x-ms-client-principal`, but the latter is easier to decode.

Neither of these things contains the email address or full name of the user.  For that, you want the results of doing a HTTP `GET /.auth/me`.  You can use the [request](https://www.npmjs.com/package/request) or [node-fetch](https://www.npmjs.com/package/node-fetch) modules to do this, but I eschew modules for straight code.  Here is my code (which is placed in the initialization section, like other functions):

{% highlight javascript %}
/**
 * Get the current auth token information from the Azure App Service Authentication
 * 
 * @param {String} authToken the authentication token
 */
function getAuthInfo(authToken, context) {
  return new Promise((resolve,reject) => {
    const options = {
      host: process.env['WEBSITE_HOSTNAME'],
      post: 443,
      path: '/.auth/me',
      method: 'GET',
      headers: { 'X-ZUMO-AUTH': authToken }
    };
    context.log(`getAuthInfo: options = ${JSON.stringify(options,null,2)}`);
    https.get(options, (response) => {
      let data = '';
      response.on('data', (chunk) => { 
        data += chunk; 
      });
      response.on('end', () => { 
        context.log(`getAuthInfo: data = ${data}`);
        let result = JSON.parse(data)[0];
        context.log(`getAuthInfo: pre-result = ${JSON.stringify(result,null,2)}`);
        let claims = {};
        result.user_claims.forEach((claim) => {
          const typ = claim.typ.split(/\//).pop();
          claims[typ] = claim.val;
        });
        context.log(`getAuthInfo: claims = ${JSON.stringify(claims,null,2)}`);
        result.claims = claims;
        delete result.user_claims;
        delete result.access_token;
        context.log(`getAuthInfo: result = ${JSON.stringify(result,null,2)}`);
        resolve(result); 
      });
    })
    .on('error', (err) => { 
      context.log(`getAuthInfo: reject ${JSON.stringify(err,null,2)}`);
      reject(err); 
    });
  });
}
{% endhighlight %}

This turns a standard HTTPS request into a promise.  My code for looking at this is as follows:

{% highlight javascript %}
  // Decode the internal authentication header that is used to pass the claims in.
  context.log(`3. Checking Authentication header`);
  if (req.headers && req.headers['x-zumo-auth'] && req.headers['x-ms-client-principal']) {
    input.principal = parsePrincipal(req.headers['x-ms-client-principal']);
    context.log(`3. Principal data = ${JSON.stringify(input.principal,null,2)}`);

    input.idp = await getAuthInfo(req.headers['x-zumo-auth'], context);
    context.log(`3. Auth Info from /.auth/me = ${JSON.stringify(input.idp,null,2)}`);
  } else {
    context.log(`3. X-ZUMO-AUTH is not there or invalid.`);
    return handleError(context, 401, "Authentication is invalid");
  }
{% endhighlight %}

I now have two objects - `input.principal`, which looks like this:

{% highlight json %}
{
  "auth_typ": "AuthenticationTypes.Federation",
  "claims": {
    "stable_sid": "sid:abcdefgc20e14193ee3e107daec9cd9",
    "nameidentifier": "sid:abcdefgb26b537824d31b4482972b",
    "identityprovider": "facebook",
    "ver": "3",
    "nbf": "1568767521",
    "exp": "1573947800",
    "iat": "1568767522",
    "iss": "https://twprodfunctions.azurewebsites.net/",
    "aud": "https://twprodfunctions.azurewebsites.net/"
  },
  "name_typ": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
  "role_typ": "http://schemas.microsoft.com/ws/2008/06/identity/claims/role"
}
{% endhighlight %}

The `input.idp` looks like this:

{% highlight json %}
{
  "expires_on": "2019-11-16T23:43:20.8656105Z",
  "provider_name": "facebook",
  "user_id": "photoadrian@outlook.com",
  "claims": {
    "nameidentifier": "1111192619711111",
    "emailaddress": "photoadrian@outlook.com",
    "name": "Adrian Hall",
    "givenname": "Adrian",
    "surname": "Hall"
  }
}
{% endhighlight %}

You can see from this that (a) the two sets of information are completely separate, and (b) you want the information in `/.auth/me` for this job.

## Working with Cosmos

The next part of the process involves working with Cosmos.  I've created a Cosmos resource in the ARM deployment, but not done anything with it thus far.  First, let's bring in the Cosmos client:

{% highlight bash %}
$> npm install --save @azure/cosmos
{% endhighlight %}

I'm using the latest production release of the Cosmos client library - v3.2.0 right now.  In the initialization section of the function, I'm going to bring that in:

{% highlight javascript %}
const { CosmosClient } = require '@azure/cosmos`;

function parseConnectionString(connectionString) {
    const parts = connectionString.split(';')
    let result = {};
    parts.forEach((v) => {
        const kv = v.split(/=(.*)/).filter(w => w !== "");
        if (kv.length >= 2) result[kv[0].toLowerCase()] = kv[1];
    });
    return result;
}

// Settings
const COSMOS_DATABASE_NAME = "tailwind";
const COSMOS_CONTAINER_NAME = "profiles";

// Get and parse the connection string
const cosmosConnectionString = process.env['COSMOSDB_CONNECTIONSTRING'];
const cosmosSettings = parseConnectionString(cosmosConnectionString);

// define a Cosmos client globally so it can be re-used
let cosmosClient = new CosmosClient(cosmosSettings);
{% endhighlight %}

Cosmos has databases.  Each database has containers, and each container has documents.  The code above creates the client to the resource, but we still need to assume that the database and container do not exist.  This is done within the Azure Function:

{% highlight javascript %}
  // Now that we have valid input, let's set up the database and container within Cosmos
  // These will throw if the database cannot be created, which in turn produces a 500 response
  context.log(`4. Creating database if it does not exist`);
  const { database } = await cosmosClient.databases.createIfNotExists({ id: COSMOS_DATABASE_NAME });
  context.log(`5. Creating container in database if not exists`);
  const { container } = await database.containers.createIfNotExists({ id: COSMOS_CONTAINER_NAME });
{% endhighlight %}

I should do something more than this if things really break, but I'm ok with returning a 500 status.  If you want to do more (like send alerts to an on-call person), wrap this in a try-catch to do it.  Now that the Cosmos service is set up properly, we can use it.  There are different things to do depending on whether the user exists or not, whether they are pending registration, and whether it's a new IdP.  Let's take a look at each one:

* If the IdP name matches a record in the database:
  * If the IdP record is PENDING, return a "code required" error.
  * If the IdP record is not PENDING, return success with the profile.
* If the IdP name does not match a record in the database, but the IdP email matches a record in the database:
  * If the IdP exists with a different name field, return a "conflict" error.
  * If the IdP does not exist, then add it to the record, return success with the profile.
* If the IdP name does not match any records in the database (including via email search):
  * If the IdP email matches the requested email, register a new account as REGISTERED, then return success with the profile.
  * If the IdP email does not match the requested email, register a new account with PENDING, send a code, and return a "code required" error.

There are probably some corner cases here.  What happens when I want to log in as an existing user with a new IdP?  What happens when I want to log in as an existing user with a new IdP that has a different email?  What happens when the IdP doesn't give me an email?  These are all additional cases that I'm going to cover.

For now, I'm going to take a look at three cases just to see the code.  These cases will represent the majority of registration calls.

1. The IdP name matches a record in the container.  This is the normal "subsequent run" version.  I'm ignoring email validation for right now.
1. The IdP name doesn't match, and the email doesn't match a record in the container.  This is the normal "first run" version.

First of all, we need to see if the IdP record matches a record in our database.  The records in the database will look like this:

{% highlgiht json %}
{
  "id": "some guid",
  "name": "My Name",
  "email": "My Email",
  "idp": {
    "facebook": {
      "id": "idp-nameidentifier",
      "email": "idp-email",
      "name": "idp-name",
      "registered": 1,
      "code": "012345"
    }
  }
}
{% endhighlight %}

The IdP information will not be returned as part of the profile.  However, we can check it during a search.  First, let's create a query and do a search:

{% highlight javascript %}
  // Do a search for the email address within the request
  const querySpec = {
      query: `SELECT * FROM Profiles p WHERE  p.idp.${input.idp.provider_name}.id = @id",
      parameters: [ { name: "@id", value: input.idp.claims.nameidentifier } ]
  };
  context.log(`6. Doing query - queryspec = ${JSON.stringify(querySpec,null,2)}`);
  const { resources: results } = await container.items.query(querySpec).fetchAll();
{% endhighlight %}

If we return one record from this search, then we've matched our first case - everything matches.  We can short-circuit a lot of checking by making this the first check.

{% highlight javascript %}
  // Case 1: If the user exists, and the IdP matches, then it's the normal run.
  if (results.length == 1) {
    context.log(`Case 1: normal operation (return profile)`);
    const profile = Object.assign({}, results[0]);
    delete profile.idp; // Don't pass IdP back
    context.res = { status: 200, body: JSON.stringify(profile) };
    return context.res;
  }
{% endhighlight javascript %}

All the other cases need to understand if the IdP email exists, so let's do that search in a similar way.

{% highlight javascript %}
  // Search for the IdP email within the request
  const emailSearch = {
    query: `SELECT * FROM Profiles p WHERE p.email = @email`,
    parameters: [ { name: "@email", value: input.idp.claims.emailaddress } ]
  };
  context.log(`6. Doing query - queryspec = ${JSON.stringify(emailSearch,null,2)}`);
  const { resources: emailResults } = await container.items.query(emailSearch).fetchAll();
  context.log(`6. Results = ${JSON.stringify(emailResults,null,2)}`);
{% endhighlight %}

We can now handle the case for when the email record does not exist, and the email in the claim matches the requested email:

{% highlight javascript %}
  if (emailResults.length == 0 && input.email === input.idp.claims.emailaddress) {
    context.log('Case 2: Normal registration with full IdP information');
    // Store the record with full registration
    const userRecord = {
      id: uuid.v4(),
      name: input.name,
      email: input.email,
      idp: {}
    };
    userRecord.idp[input.idp.provider_name] = {
      id: input.idp.claims.nameidentifier,
      email: input.idp.claims.emailaddress,
      name: input.idp.claims.name || input.name,
      registered: 1
    };
    context.log(`Registering: ${JSON.stringify(userRecord,null,2)}`);
    const { resource: profile } = await container.items.create(userRecord);
    context.log(`Registered: profile = ${JSON.stringify(profile,null,2)}`);

    // Return the stored record
    context.res = { status: 200, body: JSON.stringify(profile) };
    return context.res;
  }
{% endhighlight %}

The Cosmos container and database operations are great because they look just like CRUD operations.  If you know the ID of the record that you want, you can just `read` it.  If you want to delete it, just do it.  In this case, I'm giving the `create` call a full record.  It returns the newly inserted record with all the metadata that is stored alongside the record.  Once I've got that, I just have to return it.

Now that I have both the returning user path and the new user path sorted, I can test with Postman.  Generate the X-ZUMO-AUTH token with the Android app, paste it into the header, and do the post with the appropriate body.  The first time through will store the record and then return it.  The second time through will read the record and return it.  It will be the same record both times.  

## The Android parts

I can now switch over to the Android part of this and update the app to deal with accessing the profile.  This is just a case of doing another HTTP request - this time to `/api/register` and dealing with the response.  Since I (currently) have all the information I need, this is easily accomplished by just looking for the 200 OK response.

## Next steps

I've filled in some of the other code paths in the registration backend, and I've got some other plans on the profile page to link accounts that I will also need to deal with.  However, this is good enough for me to progress now.  You can find the code here:

* [Backend](https://github.com/adrianhall/tailwind-photos-backend/tree/blog-9)
* [Android app](https://github.com/adrianhall/tailwind-photos-for-android/tree/blog-9)

I did find a bug while researching this blog.  Azure App Service Authentication requires that the Azure Active Directory package use a JWT for the access token.  When using the MSAL library with non-organizational accounts (such as my personal account), the access token is "something else" that isn't a JWT, thus you get an unauthenticated request from the `/.auth/me/aad` login.  The solution to this (at least temporarily) is to use the Microsoft Authentication instead of Azure Active Directory.

In the next step, I'm going to start work on the main functionality of the app.  Until I've got something to share, that's it for now!


