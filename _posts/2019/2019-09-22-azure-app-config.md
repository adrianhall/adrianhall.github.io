---
title: "Using Azure App Configuration for Remote Config with Android"
categories:
  - Android
tags:
  - Kotlin
  - "Azure App Configuration"
---

I've been playing with a new app recently.  I decided I needed some support from the cloud around feature flags (turning on and off features for specific people so I can test things) and for remote configuration.  Fortunately, Azure has a service in preview - [App Configuration](https://azure.microsoft.com/en-us/services/app-configuration/) - and it does both of these things.  There are preview libraries to go along with it for .NET, Java 8, JavaScript, and Python.

Sadly, there is no library for Android Java.

Fortunately, we can fix that!  Let's take a look at how I did it.

## Creating an App Configuration resource

First step is to create an App Configuration resource.  Log on to the [Azure portal](https://portal.azure.com), create yourself a resource group, then create the App Configuration resource - just like any other resource within Azure.  You'll have to give it a name and select a region, but that's it.

Now, let's put some data within the parameter store.  To do this:

* Select your App Configuration resource
* Select **Configuration explorer**
* Click **Create**
* Enter a key and a value.  If you want (and I suggest it), also give them a label.  I'm using _production_ for my label. 
* Then click **Apply**
* Repeat the create loop for as many settings as you want.
* Once done, select **Access keys**, then select **Read-only keys**.
* Make a note of the _Endpoint_, _Id_, and _Secret_.

I've created a couple of settings here:

![]({{ site.baseurl }}/assets/images/2019/2019-09-22-image1.png)

## Configure the Android project

First step is to ensure that the code we write will be able to access the settings necessary to connect to the remote configuration service.  To do this, I've added them to the `strings.xml` file:

{% highlight xml %}
<resources>
    <string name="app_name">Tailwind Photos</string>

    <string name="azure_appconfig_endpoint">https://myconfig.azconfig.io</string>
    <string name="azure_appconfig_accesskey">ABCD-l4-s0:M0/ER+X/4rtkKabcdv9u</string>
    <string name="azure_appconfig_secret">abcdefghvK9VnntswIRL+v2MCaY7eiFon5pLTpbj8=</string>
    <string name="azure_appconfig_label">production</string>
</resources>
{% endhighlight %}

The actual values will come from the **Access keys** page of your resource.  I've also created a settings data class to hold these values so they are easier to pass around:

{% highlight xml %}
package com.tailwind.app.photos.services.azureappconfiguration

data class AzureAppConfigurationSettings(
    val endpoint: String,
    val accessKey: String,
    val accessSecret: String
)
{% endhighlight %}

To communicate with the backend resource, I'm going to use [OkHttp](https://square.github.io/okhttp/).  The configuration comes back with just one call (unless you have a large number of settings - in which case, you need to think a little bit more about the process of remote configuration).  I don't need to communicate with a large number of REST endpoints, so Retrofit (as an example) is overkill in this case.
I've added the following to my dependencies:

{% highlight gradle %}
implementation "com.squareup.okhttp3:okhttp:${versions.okhttp}",
implementation "com.squareup.okhttp3:logging-interceptor:${versions.okhttp}"
{% endhighlight %}

The current version of OkHttp is `4.2.0`.

## Create the App Configuration Service

As always when communicating with a remote service, I always create a service class that does the final leg of communication with the service.  The methods in the service class correspond 1:1 with the REST methods on the service.  I also delegate all the "protocol" level stuff (like authorization, logging, required headers) to an interceptor to make the code for the service class as simple as possible.

In this case, I have an `AzureAppConfigurationService` as follows:

{% highlight kotlin %}
package com.tailwind.app.photos.services.azureappconfiguration

import android.content.Context
import com.tailwind.app.photos.R
import com.tailwind.app.photos.services.AzureServiceException
import okhttp3.*
import okhttp3.logging.HttpLoggingInterceptor
import org.json.JSONArray
import org.json.JSONObject
import timber.log.Timber

class AzureAppConfigurationService(context: Context) {
    private val settings: AzureAppConfigurationSettings
    private val label: String = context.resources.getString(R.string.azure_appconfig_label)

    /**
     * OkHttp Client for this service
     */
    private val client: OkHttpClient

    init {
        Timber.tag(javaClass.simpleName)

        settings = AzureAppConfigurationSettings(
            context.resources.getString(R.string.azure_appconfig_endpoint),
            context.resources.getString(R.string.azure_appconfig_accesskey),
            context.resources.getString(R.string.azure_appconfig_secret)
        )

        val loggingInterceptor = HttpLoggingInterceptor(object : HttpLoggingInterceptor.Logger {
            override fun log(message: String) {
                Timber.tag("OkHttp").d(message)
            }
        })
        loggingInterceptor.level = HttpLoggingInterceptor.Level.BODY

        client = OkHttpClient.Builder()
            .addInterceptor(AzureAppConfigurationInterceptor(settings))
            .addInterceptor(loggingInterceptor)
            .build()
    }

    /**
     * Read the configuration from Azure App Configuration
     *
     * @return {Map<String,String>} key-value pairs of configuration
     */
    fun readConfiguration(): Map<String,String> {
        try {
            val kvEndpoint = "${settings.endpoint}/kv?label=${label}"

            val request = Request.Builder()
                .url(kvEndpoint)
                .build()

            val response = client.newCall(request).execute()
            if (!response.isSuccessful) {
                throw AzureServiceException("Bad response from Azure App Configuration service")
            }
            val jsonObject = JSONObject(response.body!!.string())
            val itemArray = jsonObject.getJSONArray("items")
            val result = HashMap<String,String>()
            for (i in 0 until itemArray.length()) {
                val item = itemArray.getJSONObject(i)
                result[item.getString("key")] = item.getString("value")
            }
            return result
        } catch (err: Exception) {
            Timber.e(err)
            throw AzureServiceException(
                "Error contacting App Configuration Service",
                err
            )
        }
    }
}
{% endhighlight %}

We can look at this in two sections:

* During initialization, I read the settings I placed into the `strings.xml` file and set up the HTTP client.  Interceptors construct a pipeline between the request (the thing you do) and the eventual communication, allowing you to adjust the request and response along the way.  In this case, there are two interceptors - one for doing all the requirements for Azure App Configuration, and one for logging the request and response.  I'm using the standard logging interceptor to send the request/response to [Timber](https://github.com/JakeWharton/timber) for the logging.
* The only endpoint I care about is the one that reads the entire configuration.  I read all the settings configured that are labelled a specific way (in this case, `production`).  This allows me to have different configuration sets for production vs. test vs. debug (or production vs. development).  It turns the response into a key-value map and returns it.  If anything goes wrong, it throws an error.

This is a pretty standard call, but where did I get the information from?  Azure does a really good job of explaining all their REST calls by [publishing the Swagger](https://github.com/Azure/azure-rest-api-specs).  This is the first place I look when discovering what a service can actually do.  In this particular case, however, the App Configuration team has published a series of [explanatory documents](https://github.com/Azure/AppConfiguration#rest-api-reference) as well.  I'm using the [key-values](https://github.com/Azure/AppConfiguration/blob/master/docs/REST/kv.md) document to determine what needs to be done.  It says if I do a `GET /kv?label=<something>`, then I'll get a JSON document back with an element `items` that contains an array of items.  Those items have a key, value, etag (to determine if it has changed), date, etc.  I'm only interested in the current key-value pair, so I extract those and return them as a map.

If anything goes wrong in the decoding, I just throw an error.  Aside from being logged, we'll see how that gets trapped later on.

## The special App Configuration interceptor

Every single service on the Internet has a method of authenticating the user and authorizing operations.  This is codified for App Configuration in their [authentication](https://github.com/Azure/AppConfiguration/blob/master/docs/REST/authentication.md) document.  I don't like to have this detail embedded with every single request, so I abstract it into an interceptor.  Let's go through the code:

{% highlight kotlin %}
package com.tailwind.app.photos.services.azureappconfiguration

import okhttp3.Headers
import okhttp3.Interceptor
import okhttp3.Request
import okhttp3.Response
import java.security.MessageDigest
import java.text.SimpleDateFormat
import java.util.*
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec

/**
 * Class for ensuring all the correct headers are on the request for Azure App Configuration
 */
class AzureAppConfigurationInterceptor(private val settings: AzureAppConfigurationSettings) : Interceptor {
    /**
     * Date formatter for the HTTP Date Header
     */
    private val dateFormatter =
        SimpleDateFormat("EEE, dd MMM yyyy HH:mm:ss 'GMT'", Locale.US).apply {
            timeZone = TimeZone.getTimeZone("GMT")
        }

    /**
     * Decoded version of the access secret
     */
    private val secret = Base64.getDecoder().decode(settings.accessSecret)

    /**
     * Hasher for the HMAC-SHA256 signature
     */
    private val hmacSigner = Mac.getInstance("HmacSHA256").apply {
        init(SecretKeySpec(secret, "HmacSHA256"))
    }

    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val dateHeader = dateFormatter.format(Date())
        val contentSHA = sha256(originalRequest.body?.toString() ?: "")

        // Compute the authorization header
        val headers = Headers.headersOf(
            "Accept", "application/vnd.microsoft.appconfig.kv+json",
            "Host", originalRequest.url.host,
            "Date", dateHeader,
            "x-ms-date", dateHeader,
            "x-ms-content-sha256", contentSHA
        )
        val authorization = authorization(originalRequest, headers)

        // Construct the new request
        val newRequest = originalRequest.newBuilder()
            .headers(headers)
            .header("Authorization", "HMAC-SHA256 $authorization")
            .method(originalRequest.method, originalRequest.body)
            .build()

        return chain.proceed(newRequest)
    }

    /**
     * Compute the authorization header
     */
    private fun authorization(request: Request, headers: Headers): String {
        val hdrs = mapOf(
            "Host" to headers["Host"],
            "x-ms-date" to headers["x-ms-date"],
            "x-ms-content-sha256" to headers["x-ms-content-sha256"]
        )
        val signedHeaders = hdrs.keys.joinToString(";")
        val hdrValues = hdrs.values.joinToString(";")

        val prefix = "Credential=${settings.accessKey}&SignedHeaders=${signedHeaders}"
        val query = if (request.url.encodedQuery != null) "?${request.url.encodedQuery}" else ""
        val stringToSign = "${request.method}\n${request.url.encodedPath}${query}\n${hdrValues}"
        val signature = Base64.getEncoder().encodeToString(hmacSigner.doFinal(stringToSign.toByteArray()))
        return "${prefix}&Signature=${signature}"
    }

    /**
     * Compute the SHA256 for some content
     */
    fun sha256(content: String): String {
        val bytes = MessageDigest.getInstance("SHA-256").digest(content.toByteArray())
        return Base64.getEncoder().encodeToString(bytes)
    }
}
{% endhighlight %}

The main method here is the `intercept` method.  It is the only method that is required to be an interceptor.  In this particular version, we are altering the request object for every single request that comes through.  We need some headers, which are added automatically.  There are two special headers:

* `x-ms-date` is a copy of the `Date` header and must be included.
* `x-ms-content-sha256` is the Base64-encoded SHA256 hash of the body of the request.

In our case, we don't have a body, so I compute the hash of the empty string.  Then, I need to compute the `Authorization` header.  This is a HMAC-SHA256 signature of a specific string, using the Id and Secret of the resource to compute the signature.  The computation is covered (in several languages - just not Kotlin) in the `authentication.md` file.  I like the Kotlin computation - it feels easier.  The only gotcha is that the header values must be in the same order as the signed headers list, and the code enforces this.

## Handling errors

Your app should not cease to function if the App Configuration service goes down, and there are many reasons you may not be able to reach the App Configuration service - throttling, network issues, and service issues all come into the picture.

If, for whatever reason, we cannot reach the service, we can use a cached version of the configuration.  For this purpose, I've got a higher level configuration service:

{% highlight kotlin %}
package com.tailwind.app.photos.configuration

import android.content.Context
import com.tailwind.app.photos.services.azureappconfiguration.AzureAppConfigurationService
import org.json.JSONObject
import timber.log.Timber
import java.io.File
import java.util.concurrent.atomic.AtomicBoolean

/**
 * The LocalConfigurationService is a read-only key-value store, implemented
 * as a set of two specific stores - the local store (on disk, provided just
 * in case there is an issue with the remote store) and the remote store.
 *
 * The local store is config.json in the application directory.
 *
 * The remote store is Azure App Configuration - when available, it overrides
 *  the local store.  It is defined by AzureAppConfiguration class, which uses
 *  a local embedded resource to configure itself.
 */
class LocalConfigurationService private constructor(context: Context) {
    private val kvStore: MutableMap<String,String> = HashMap()

    init {
        Timber.tag(javaClass.simpleName)

        // Read the local file if it exists
        Timber.d("Looking for local config.json")
        val file = File(context.filesDir, CONFIG_FILE)
        if (file.exists()) {
            Timber.d("Found local config.json")
            val localJsonString = file.readText(Charsets.UTF_8)
            val jsonObject = JSONObject(localJsonString)
            val iterator = jsonObject.keys()
            Timber.d("Adding key-values from local config.json into combined map")
            while (iterator.hasNext()) {
                val key = iterator.next()
                kvStore[key] = jsonObject.getString(key)
                Timber.d("KV(${key} = ${kvStore[key]}")
            }
            Timber.d("Finished loading local config.json")
        } else {
            Timber.d("Did not find local config.json")
        }

        val remoteConfigLoaded = AtomicBoolean(false)
        try {
            Timber.d("Looking for remote config");
            val appConfigService = AzureAppConfigurationService(context)
            Timber.d("Reading remote config from remote service")
            val remoteMap = appConfigService.readConfiguration()
            Timber.d("Merging remote service into local config")
            kvStore.putAll(remoteMap)       // Add the remote map into the local map
            remoteConfigLoaded.set(true)
        } catch (err: Exception) {
            Timber.e(err)

            if (!file.exists()) {
                // We have never actually reached the remote store before, so we are
                // kind of screwed when it comes to configuring ourselves.  We need
                // to throw an error instead.
                throw UninitializedConfigurationStore()
            }

            // Otherwise, swallow this error and continue with the old configuration
        }

        // At the end, write out the new configuration if we loaded a new remote config
        if (remoteConfigLoaded.get()) {
            writeLocalConfiguration(context)
        }
    }

    private fun writeLocalConfiguration(context: Context) {
        Timber.d("Writing new local config.json")
        val jsonObject = JSONObject(kvStore as Map<*, *>)
        val file = File(context.filesDir, CONFIG_FILE)
        file.writeText(jsonObject.toString(2), Charsets.UTF_8)
        Timber.d("Finished writing new local config.json")
    }

    companion object {
        private const val CONFIG_FILE = "config.json"
        private lateinit var instance : LocalConfigurationService
        private val initialized = AtomicBoolean()

        /**
         * Initialize the local configuration store.  This must be called before
         * the key-value store is used.
         *
         * @param {Context} context the application context
         */
        fun initialize(context: Context) {
            if (!initialized.get()) {
                instance = LocalConfigurationService(context)
                initialized.set(true)
            }
        }
    }
}
{% endhighlight %}

This is a singleton object, which needs to call `LocalConfigurationStore.initialize(context)` to retrieve the configuration into memory.  It constructs it by:

* Reading the configuration from the cached `config.json` file.  It won't exist the first time through.
* Reading from remote configuration, overwriting the local configuration.
* If remote configuration was successful, then write out the cached `config.json` version.

If remote configuration (i.e. the read from the App Configuration service) was not successful for any reason, then the latest cached copy of the configuration is used instead.  The only time this fails is if there is no cached copy.  We haven't properly initialized the configuration yet.  Adding in retry logic with incremental back-off would alleviate this problem as well unless the user had no network signal at all.

I don't want to be doing `LocalConfigurationStore.instance.kvStore.whatever()` all the time, especially since the service is read-only, so I added common `get()` operations to the companion object to do that for me:

{% highlight kotlin %}
    companion object {
        /**
         * Retrieve the value of a key.  Returns null if the key does not exist.
         *
         * @param {String} key the name of the key to retrieve
         * @return the value of the key, or null if the key does not exist
         */
        fun get(key: String): String? {
            if (!initialized.get()) {
                throw UninitializedConfigurationStore()
            }

            return instance.kvStore[key]
        }

        /**
         * Retrieve the value of a key.  Returns the default value provided if the
         * key does not exist.
         *
         * @param {String} key the name of the key to retrieve
         * @param {String} defaultValue the default value to return if the key does not exist
         * @return the value of the key, or the default value
         */
        fun optGet(key: String, defaultValue: String): String {
            if (!initialized.get()) {
                throw UninitializedConfigurationStore()
            }

            return instance.kvStore.getOrDefault(key, defaultValue)
        }

        /**
         * Returns true if the key exists in the store
         *
         * @param {String} key the name of the key to check
         * @return true if the key exists
         */
        fun containsKey(key: String): Boolean {
            if (!initialized.get()) {
                throw UninitializedConfigurationStore()
            }

            return instance.kvStore.containsKey(key)
        }
    }
}
{% endhighlight %}

This allows me to do `LocalConfigurationStore.get("auth.fb.appkey")`, for example, to retrieve a specific key.  The rest of my code is likely to know what keys they need.

## Future improvements

Aside from the afore mentioned incremental back-off and retry logic, I might also distribute a `config.json` file in the `res/raw` directory.  This can be read through `resources.openRawResource()`.  The point?  It allows me to distribute a "default" configuration and use that for the condition of "I've not downloaded the current configuration yet" situation.


