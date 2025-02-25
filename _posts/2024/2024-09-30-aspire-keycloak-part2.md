---
title:  "Customizing Keycloak with Aspire - Part 2"
date:   2024-09-30
categories:
    - Web
tags:
    - aspnetcore
    - aspire
    - keycloak
---

In [my last article]({% post_url 2024/2024-09-26-aspire-keycloak-part1 %}), I introduced the Keycloak identity service and showed how the development version can be easily integrated into a project.  Development services get you started quickly and allow you to defer the details until later on.  At some point, however, you need to take control of your service and start working towards production.  There are a number of things that the development version of the Keycloak identity service doesn't do that you need in production.  These include SSL/TLS support, database storage, and proper configuration of the flows.

In short, you need to be able to customize the identity service in code.

Fortunately, Keycloak has many options for configuring the identity service via the container.  You do need to build a new version of the container to do this.  In this article, I'm going to show you how you can go from the original use of Keycloak to a custom container that you control.

## Using a custom Dockerfile

Aspire (and most other complex applications) rely on extension methods to hide the details of how a service is configured, allowing you to concentrate on what it is actually providing for you.  Keycloak is no different.  Let's take the code I introduced in the last article:

```csharp
var keycloak = builder
  .AddKeycloakContainer("keycloak")
  .WithDataVolume()
  .WithImport("./KeycloakConfiguration/Test-realm.json")
  .WithImport("./KeycloakConfiguration/Test-users-0.json");

var realm = keycloak.AddRealm("Test");
```

This code is in the `Program.cs` file of the AppHost project and sets up the Keycloak container.  You can write a custom Dockerfile and use it to build a new version of the service from that Dockerfile.  For example, the [Keycloak documentation](https://www.keycloak.org/server/containers) contains a guide for running as a container with an example Dockerfile.  I placed a similar Dockerfile within the `./KeycloakConfiguration` directory:

```docker
FROM quay.io/keycloak/keycloak:latest AS builder

# Enable health and metrics support
ENV KC_HEALTH_ENABLED=true
ENV KC_METRICS_ENABLED=true

WORKDIR /opt/keycloak

# for demonstration purposes only, please make sure to use proper certificates in production instead
RUN keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 \
	-dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" \
	-keystore conf/server.keystore

RUN /opt/keycloak/bin/kc.sh build

FROM quay.io/keycloak/keycloak:latest
COPY --from=builder /opt/keycloak/ /opt/keycloak/

ENV KC_HOSTNAME=localhost

ENTRYPOINT ["/opt/keycloak/bin/kc.sh"]
```

The Aspire hosting model uses Keycloak v24.0.3.  The version is embedded in the image tag that is used to download the container.  This Dockerfile uses the latest tag, which is v25.0.6. Now that I have the Dockerfile, I need to update the resource definition in `Program.cs`:

```csharp
var keycloak = builder
    .AddKeycloakContainer("keycloak")
    .WithDockerfile("./KeycloakConfiguration", "Dockerfile")
    .WithDataVolume()
    .WithImport("./KeycloakConfiguration/Test-realm.json")
    .WithImport("./KeycloakConfiguration/Test-users-0.json");
```

When you run this, you can see the container image being built.  Log into the console and check the **Server info** tab to see the version.

## Adding a HTTPS endpoint

Let's start providing remedies for the list of deficiencies that I mentioned in [my last article]({% post_url 2024/2024-09-26-aspire-keycloak-part1 %}).  First up, I wanted to ensure logins were handled via a secure HTTPS request.  The following line in the Dockerfile is of interest:

```docker
RUN keytool -genkeypair -storepass password -storetype PKCS12 -keyalg RSA -keysize 2048 \
	-dname "CN=server" -alias server -ext "SAN:c=DNS:localhost,IP:127.0.0.1" \
	-keystore conf/server.keystore
```

This creates a certificate for development purposes and allows us to expose the HTTPS endpoint.  This is another update to the definition in the `AppHost` project:

```csharp
var keycloak = builder
    .AddKeycloakContainer("keycloak")
    .WithDockerfile("./KeycloakConfiguration", "Dockerfile")
    .WithHttpsEndpoint(port: 8443, targetPort: 8443, name: "https")
    .WithDataVolume()
    .WithImport("./KeycloakConfiguration/Test-realm.json")
    .WithImport("./KeycloakConfiguration/Test-users-0.json");
```

When you start the service back up, you will note that the additional endpoint has been added.  Unfortunately, there isn't a way of disabling the HTTP port unless you unwind the extension method for `.AddKeycloakContainer()` which I don't recommend.

You will want to get a certificate from somewhere like [Let's Encrypt](https://letsencrypt.org/).  You can [read about the process here](https://cloudspinx.com/run-keycloak-in-docker-or-podman-with-lets-encrypt-ssl/).  Ideally, you would be storing the certificate in [Azure Key Vault](https://learn.microsoft.com/azure/key-vault/certificates/quick-create-cli).  However, the setup for this is beyond this article and involves additional changes to the Dockerfile.

## Updating the test realm

As part of the definition, I've included a test realm that is minimally configured.  This also allows me to define a new realm.  Log onto the Keycloak service, then use the drop-down in the left hand menu and select **Create realm**.  You'll need to give it a name, then click on **Create**.  You can configure the new realm in any way that you want.  Some things to do:

* Configure all the **Realm settings**.  The **Login** tab allows you to set up user registration, forgotton password flows, and email confirmation settings.
* Configure the **Identity providers** to add support for social media providers like Facebook, Google, LinkedIn, and Microsoft accounts.
* Create your **Clients**.

When you are happy with your configuration (and your application is working with the updates), you can export your definition.

* Go to **Realm settings**.
* In the top right corner, use the **Action** pull-down to select **Partial export**.
* Choose what you want to export (likely everything), then select **Export**.

The file `realm-export.json` is generated.  This can be used in place of `Test-realm.json` with the `.WithImport()` method.  You can (and should) remove the `Test-users-0.json` import when running in production.  You can use `builder.ExecutionContext.IsRunMode` for this.

```csharp
if (builder.ExecutionContext.IsRunMode)
{
    keycloak.WithImport("./KeycloakConfiguration/Test-users-0.json");
}
```

## Final thoughts

I've got two concerns left to deal with:

* Storing the identity data in a SQL database like PostgreSQL.
* Adding a theme so that the web forms look like a part of the application.

Aside from these topics, you should be able to configure the Keycloak service for your application and integrate it fully into your application using the documentation from Keycloak itself together with a solid understanding of OAuth2.0 and OIDC principals.  Some of this is relatively complex, so I'll cover the remaining topics at a later date.

It also looks like the Keycloak hosting model for Aspire is going to become an official part of Aspire soon.  You can [check out the code here](https://github.com/dotnet/aspire/tree/main/src/Aspire.Hosting.Keycloak).

## Further reading

* [The project so far][github]
* [Keycloak]
* [.NET Aspire][Aspire] (check out [Discord](https://discord.com/channels/732297728826277939/759125320505884752) and [samples](https://github.com/dotnet/aspire-samples))
* [Using Dockerfile with .NET Aspire][dockerfile]

<!-- Links -->
[Aspire]: https://learn.microsoft.com/dotnet/aspire/get-started/aspire-overview
[Keycloak]: https://www.keycloak.org/
[Oleksii Nikiforov]: https://github.com/NikiforovAll
[github]: https://github.com/adrianhall/samples/tree/0930/keycloak
[dockerfile]: https://learn.microsoft.com/dotnet/aspire/app-host/withdockerfile