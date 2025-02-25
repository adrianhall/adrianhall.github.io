---
title:  "Customizing Keycloak with Aspire - Part 3"
date:   2024-10-04
categories:
    - Web
tags:
    - aspnetcore
    - aspire
    - keycloak
---

This is the last article in a trio of articles about incorporating Keycloak into .NET Aspire applications.  Thus far, I've [covered the basics]({% post_url 2024/2024-09-26-aspire-keycloak-part1 %}) and [some common customizations]({% post_url 2024/2024-09-30-aspire-keycloak-part2 %}).  This part covers what you need to do if you want to store the identity information within a PostgreSQL database.  I've chosen PostgreSQL for this because it seems to be the database of choice for Aspire when you want to run things locally and in the cloud and also because the PostgreSQL database driver is included by default in the Keycloak container.  If you want a different database, you likely need to install a JDBC driver in the container during build time.

## Create a database

Start by updating the `AppHost` configuration to include a database.  First, I need to add two parameters that represent the username and password for the database.  Normally, `.AddPostgres()` will use username `postgres` and generate a password. The connection string that is generated is not compatible with JDBC connections, so I need to actually specify a username and password so I can construct the right JDBC connection string.

> **Set up your database properly in production**<br/>
> You should never use the database administrator username and password to connect to the database.  I'm doing this for ease of use and to avoid setting up the database with scripts.  Check out the [database connections sample](https://github.com/dotnet/aspire-samples/tree/main/samples/DatabaseContainers) to see how to run a SQL script to configure your database.  You should use this mechanism to create a user that only has access to your database; then use that user!
> {: .notice--warning}

Add the following snippet to the `AppHost/appsettings.json` file:

{% highlight json %}
{
  "Parameters": {
    "username": "postgres",
    "password": "Pa@sw0rd"
  }
}
{% endhighlight %}

> **Don't make passwords too complex**<br/>
> When using containers, passwords are inevitably introduced as environment variables which are subject to shell parsing rules.  Certain characters (back-slash, back-tick, dollar, exclamation point) get special treatment.  Avoid those special characters.
> {: .notice-warning}

Then, in the `AppHost/Program.cs`:

{% highlight csharp %}
var username = builder.AddParameter("username");
var password = builder.AddParameter("password");

var dbserver = builder
    .AddPostgres("postgres", username, password)
    .WithEnvironment("POSTGRES_DB", "identity")
    .WithPgAdmin();

var identitydb = dbserver.AddDatabase("identity");
{% endhighlight %}

**AddDatabase() does not create a database.**  

You need to create a database as a separate action.  One way to do this is to add the `.WithEnvironment()` line - the container will create the database that is specified by the `POSTGRES_DB` environment variable if it exists.  This doesn't help you if you have multiple databases to create, but it works in this scenario.  See the [database connections sample](https://github.com/dotnet/aspire-samples/tree/main/samples/DatabaseContainers) for other ways to run SQL on your database during the creation process.

## Add Postgres to the Dockerfile

I have to update the `Dockerfile` to support PostgreSQL:

{% highlight docker %}
FROM quay.io/keycloak/keycloak:latest AS builder

# Enable health and metrics support
ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true

# Configure a database vendor
ENV KC_DB=postgres

WORKDIR /opt/keycloak

# for demonstration purposes only, please make sure to use proper certificates in production instead
RUN keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 \
	-dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" \
	-keystore conf/server.keystore

RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:latest
COPY --from=builder /opt/keycloak/ /opt/keycloak/

# change these values to point to a running postgres instance
ENV KC_DB=postgres
ENV KC_DB_USERNAME=keycloak
ENV KC_DB_PASSWORD=keycloak
ENV KC_DB_URL=jdbc:postgresql://localhost:5432/identity

# Other settings you may need to set.
ENV KC_HOSTNAME=localhost

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
{% endhighlight %}

I need to specify the `KC_DB` environment variable during the build process so that the PostgreSQL database driver is included in the build.  If you had a different (not included) driver, you would include it at this point by copying it to the right place in /opt/keycloak on the build container.  I also need to specify three additional environment variables to configure the PostgreSQL database on the running container.

## Get the JDBC expression for the database

Naturally, Aspire focuses on dotnet connection strings.  Keycloak is written in Java, so it requires JDBC connection strings.  They are very different.  Here is an example for the JDBC connection string:

{% highlight text %}
jdbc:postgresql://host.docker.internal:60397/identity
{% endhighlight %}

When running in "run mode" (i.e. docker desktop), the hostname must be "host.docker.internal" even though the endpoint hostname will be "localhost".  In production mode, I would hope you are using a managed database.  The endpoint name will be provided by the managed database in that case and you can just use it.

To help build the right reference expression, I created a static class:

{% highlight csharp %}
internal static class JdbcExpression
{
    internal static ReferenceExpression Create(IResourceBuilder<PostgresDatabaseResource> dbResourceBuilder)
    {
        PostgresDatabaseResource dbResource = dbResourceBuilder.Resource;
        PostgresServerResource dbServer = dbResource.Parent;
        EndpointReference endpoint = dbServer.PrimaryEndpoint;

        if (dbResourceBuilder.ApplicationBuilder.ExecutionContext.IsRunMode)
        {
            // In run mode, use host.docker.internal instead of localhost
            return ReferenceExpression.Create($"jdbc:postgresql://host.docker.internal:{endpoint.Property(EndpointProperty.Port)}/{dbResource.Name}");
        }
        else
        {
            // In production mode, assume the endpoint hostname is correct
            return ReferenceExpression.Create($"jdbc:postgresql://{endpoint.Property(EndpointProperty.Host)}:{endpoint.Property(EndpointProperty.Port)}/{dbResource.Name}");
        }
    }
}
{% endhighlight %}

I can pass in a reference to a database with this.  The code looks up the server reference and the endpoint reference before constructing a reference expression for me.  Note that the argument to the `ReferenceExpression.Create()` method is an interpolated string.  You can't use a static or constant string here as it is interpolated when the orchestration happens (i.e. not immediately).

## Update the Keycloak reference

Now that I've got a way of generating a JDBC reference, I can update the `AppHost/Program.cs` again - this time to configure Keycloak so that it uses the database I've created:

{% highlight csharp %}
var keycloak = builder
    .AddKeycloakContainer("keycloak")
    .WithDockerfile("./KeycloakConfiguration", "Dockerfile")
    .WithHttpsEndpoint(port: 8443, targetPort: 8443, name: "https")
    .WithDataVolume()
    .WithEnvironment("KC_DB_USERNAME", username)
    .WithEnvironment("KC_DB_PASSWORD", password)
    .WithEnvironment("KC_DB_URL", JdbcExpression.Create(identitydb))
    .WithImport("./KeycloakConfiguration/Test-realm.json");

if (builder.ExecutionContext.IsRunMode)
{
    keycloak.WithImport("./KeycloakConfiguration/Test-users-0.json");
}
{% endhighlight %}

One thing that bit me here: the `.WithEnvironment()` values are not strings and can't be treated as strings.  They are references that are computed when the orchestration happens.

Now that everything is done, fire up the application and watch the Keycloak logs (which you can do within the Aspire dashboard) to make sure the database gets created.  Once you see that, everything should "just work".

## The problems I ran into

I ran into a lot of problems when I was building this:

### Problem: could not access database service with PgAdmin

The problem here was surprising.  The database username is expected to be "postgres".  Even if you set the database username as a parameter, PgAdmin doesn't pick it up.  Also, I found passwords that contain characters commonly interpreted by the shell are a problem.  This includes the slash, back-slash, dollar, and exclamation point.

### Problem: database not created

Once I got connectivity to the database service, my Keycloak server was still not coming up.  I started the root cause at PgAdmin and discovered that `.AddDatabase()` didn't create the database.  It was a quick fix buried in a comment to an issue that solved this problem.  However, the Aspire team also have [a set of samples](https://github.com/dotnet/aspire-samples).  The fix was also included as an aside in one of the samples.

### Problem: Keycloak cannot communicate with the database

Finally, I was using `jdbc:postgresql://localhost:60397/identity` as my database JDBC URL.  The localhost refers to the "current container".  I needed to switch to "host.docker.internal".  I found this out by looking at what connection string was being used by PgAdmin.

### Problem: Changes do not take effect in the database

One of the problems I encountered was due to the early use of `.WithDataVolume()` to persist the database.  Once the database was created, it doesn't get re-created.  That means you can't just change the password in settings - you have to change it on the data volume (via the database PgAdmin) as well.  I didn't realize this and spent a wasted afternoon diagnosing a connectivity problem.  If you do major configuration changes, open up Docker Desktop and delete the volumes you have created that may be affected.  They will be re-created when you run the application again.

## Final thoughts

Aspire is young - really young.  With young products, the documentation and community resources (like Stack Overflow) are not always available and there aren't enough samples to drive AI copilot functionality to help you learn.  You have to dig through similar problems and read the notes, comments, and samples to find potential fixes to your problems.  Don't be afraid to reach out to people that seem to know what they are talking about.  The JDBC expression fix was produced based on a response from one of the Aspire engineers who is active in the [Aspire repository](https://github.com/dotnet/aspire).

Now, back to my project!

## Further reading

* [The final sample](https://github.com/adrianhall/samples/tree/1004/keycloak)
* [Keycloak](https://www.keycloak.org/documentation)
* [.NET Aspire](https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview)
  * Check out [Discord](https://discord.com/channels/732297728826277939/759125320505884752) and [samples](https://github.com/dotnet/aspire-samples)
* [Aspire samples](https://github.com/dotnet/aspire-samples)