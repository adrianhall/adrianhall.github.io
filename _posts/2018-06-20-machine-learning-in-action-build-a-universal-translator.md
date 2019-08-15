---
title: "Machine Learning in Action: Build a Universal Translator App for Android with Kotlin"
categories:
  - Android
tags:
  - Kotlin
  - "Machine Learning"
---

I’ve started playing with some of the machine learning APIs that AWS provides. One of these is [Amazon Translate] — a service that translates from English to a variety of other languages. So I thought I would make an app that uses the on-board speech recognition of an Android app, then translates it to a new language.

## What you will need

The requirements for this project are fairly simple. You will need an AWS account (which you can [get for free](https://aws.amazon.com/free)), an Android phone (I had problems recording audio on the emulator so I recommend a real device), and Android Studio (which is a free download).

## What we will do

We are going to implement two distinct pieces in this tutorial. Firstly, we are going to create a UI that records some audio and then converts it to text. Then we will send that text to the Amazon Translate service and display the result.

## Speech to text

Start by creating a new Android project. Make sure you select an appropriate API level (which will depend on your device). Speech recognition was added in API level 8, so you have plenty of room to use older devices. Ensure you add Kotlin support as well. Select the blank template (Empty activity) to start.

Add the following permissions to the `AndroidManifest.xml`:

{% highlight xml %}
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
{% endhighlight %}

The first one allows us to access the microphone. The other two allow us to send the result to the Internet. By including `ACCESS_NETWORK_STATE`, you can add the ability to detect whether you are connected to the Internet so that the app doesn’t crash when the network is unavailable.

Now, let’s take a look at the `res/layout/activity_main.xml` file:

{% highlight xml %}
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".ui.MainActivity">

    <Button
        android:enabled="false"
        android:id="@+id/btn_start_recognizer"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:layout_marginStart="8dp"
        android:layout_marginTop="24dp"
        android:text="@string/btn_start_recognizer"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <TextView
        android:id="@+id/recorded_speech"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:layout_marginStart="8dp"
        android:layout_marginTop="24dp"
        android:text="Recorded text will appear here"
        android:textSize="20sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/btn_start_recognizer" />

    <TextView
        android:id="@+id/translated_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginEnd="8dp"
        android:layout_marginStart="8dp"
        android:layout_marginTop="24dp"
        android:text="Translated text will appear here"
        android:textSize="20sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/recorded_speech" />
</android.support.constraint.ConstraintLayout>
{% endhighlight %}

Layouts don’t get much simpler than this. There is a button at the top to activate the recording, and two text boxes — the first will show the text that we heard and the second will show the translated text.

Now let’s take a look at the `MainActivity.kt` file:

{% highlight kotlin %}
package com.shellmonger.apps.speechtranslator.ui

import android.content.Intent
import android.support.v7.app.AppCompatActivity
import android.os.Bundle
import android.speech.RecognizerIntent
import android.speech.SpeechRecognizer
import android.util.Log
import com.shellmonger.apps.speechtranslator.R
import com.shellmonger.apps.speechtranslator.TAG
import kotlinx.android.synthetic.main.activity_main.*
import java.lang.Exception

/**
 * The main activity is the first (and only) activity that is displayed for this app
 */
class MainActivity : AppCompatActivity() {
    companion object {
        const val REQUEST_SPEECH_RECOGNIZER = 1000
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        initializeClient()

        // Wire up the button to start recognizing the speech
        btn_start_recognizer.setOnClickListener {
            val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
                putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
            }
            startActivityForResult(intent, REQUEST_SPEECH_RECOGNIZER)
        }

        // The button is (by default) disabled, so you can't click it.  Enable it if the
        // isRecognitionAvailable() supported
        if (SpeechRecognizer.isRecognitionAvailable(this)) {
            btn_start_recognizer.isEnabled = true
        } else {
            Log.i(TAG, "Speech Recognition is not available")
        }
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == REQUEST_SPEECH_RECOGNIZER) {
            if (resultCode == RESULT_OK) {
                data?.let {
                    val results = data.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)
                    recorded_speech.text = results[0]
                }
            }
        }
    }

    private fun initializeClient() {
    }
}
{% endhighlight %}

The button to initiate recording is only enabled if speech recognition is available. If that happens, then we call out to the speech recognizer service via an intent. This will pop up a small dialog. At this point, the user speaks. When the user stops speaking, the service will return the text using `onActivityResult()`. At that point, we update the UI to display the results.

That was so easy to get started! You can actually run this app and see the speech to text working.

There are a couple of caveats here. The most important is that Android calls out to a Google service to complete the process. As of API level 23, there is an additional option for the service called `EXTRA_PREFER_OFFLINE` that can be used to indicate you prefer to do this offline. Use it like this:

{% highlight kotlin %}
val intent = Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH).apply {
    putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL,
        RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
    putExtra(RecognizerIntent.EXTRA_PREFER_OFFLINE, true)
}
startActivityForResult(intent, REQUEST_SPEECH_RECOGNIZER)
{% endhighlight %}

## Set up text translation service

Before you can use the text translation service, you need to set it up. I’ve got an [AWS CloudFormation template](https://gitlab.com/adrianhall/speech-translator-app/blob/master/service/SpeechTranslator.yaml) for this purpose, which I install using the AWS command line. First, copy it to an S3 bucket. I have a scratchpad S3 bucket that I use for this sort of thing:

```
aws s3 sync service s3://my-scratchpad/cf
```

The YAML file is located in [the service directory of my GitLab repository](https://gitlab.com/adrianhall/speech-translator-app/tree/master/service).

```
aws cloudformation create-stack --stackname speech-translator --template-url https://s3-us-west-2.amazonaws.com/my-scratchpad/cf/SpeechTranslator.yaml --capabilities CAPABILITY_NAMED_IAM
```

The translate service doesn’t need anything special, but it does need an IAM role to approve the request. The CloudFormation template sets up an unauthenticated IAM role, then associates that unauthenticated IAM role to a newly created Amazon Cognito identity pool. The identity pool will give us temporary credentials to access the Amazon Translate service later on.

## Create an AWS connection in the app

The app needs to know where and how to connect to the Amazon Translate API. For this, I created a JSON file in `res/raw/aws.json` with the information from my CloudFormation stack:

{% highlight json %}
{
  "region": "us-west-2",
  "accountId": "01234567890",
  "identityPoolId": "us-west-2:IDPOOL_GUID",
  "unauthRoleArn": "arn:aws:iam::01234567890:role/speech-translator-UnAuthenticatedRole-ABCDEFHJIJKL",
  "authRoleArn": "arn:aws:iam::01234567890:role/speech-translator-AuthenticatedRole-MNOPQRSTUV"
}
{% endhighlight %}

> Don't check in the `aws.json` file.  It contains secrets!

To get the various values, take a look at the **Outputs** section of the CloudFormation stack. Once the stack is finished, you can use the following command:

```
aws cloudformation describe-stack --stack-name speech-translator
```

This will show you the details for the named stack, which includes the three values that are relatively harder to obtain. The accountId is the 12 digit number for your account — available in the top banner of the AWS console.

To read this file, I’ve got a model:

{% highlight kotlin %}
package com.shellmonger.apps.speechtranslator.models

import android.content.Context
import org.json.JSONObject
import java.io.BufferedReader
import java.io.InputStreamReader

class AWS {
    var accountId: String = ""
    var region: String = ""
    var identityPoolId: String = ""
    var unauthRoleArn: String = ""
    var authRoleArn: String = ""

    companion object {
        fun fromJson(json: String): AWS {
            val o = JSONObject(json)
            val model = AWS().apply {
                accountId = o.getString("accountId")
                region = o.getString("region")
                identityPoolId = o.getString("identityPoolId")
                unauthRoleArn = o.getString("unauthRoleArn")
                authRoleArn = o.getString("authRoleArn")
            }
            return model
        }

        fun fromResource(context: Context, resourceId: Int): AWS {
            val inputStream = context.resources.openRawResource(resourceId)
            val inputReader = BufferedReader(InputStreamReader(inputStream))
            val json = inputReader.readText()
            inputReader.close()
            inputStream.close()
            return fromJson(json)
        }
    }
}
{% endhighlight %}

This is a basic model for the five values we need to configure the AWS Mobile SDK. I’ve added two helper methods for converting from a JSON string to the object, and for reading the JSON string from a resource.

Next, let’s add the AWS Mobile SDK for Android libraries. In the app-level `build.gradle` file, add the following dependencies:

{% highlight gradle %}
dependencies {
    def aws_version = '2.6.29'

    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation"org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    implementation 'com.android.support:appcompat-v7:27.1.1'
    implementation 'com.android.support.constraint:constraint-layout:1.1.2'

    // AWS SDK
    implementation "com.amazonaws:aws-android-sdk-core:$aws_version"
    implementation "com.amazonaws:aws-android-sdk-auth-core:$aws_version"
    implementation "com.amazonaws:aws-android-sdk-translate:$aws_version"
}
{% endhighlight %}

We’re now ready to configure the AWS translate client. You may have noticed the blank `initializeClient()` in the earlier code. This is now going to be replaced:

{% highlight kotlin %}
private var translateClient: AmazonTranslateAsyncClient? = null

/**
  * Initialize the AWS client from the aws.json file
  */
private fun initializeClient() {
    val config = AWS.fromResource(this, R.raw.aws)
    val credentialsProvider = CognitoCredentialsProvider(
            config.accountId,
            config.identityPoolId,
            config.unauthRoleArn,
            config.authRoleArn,
            Regions.fromName(config.region)
    )
    translateClient = AmazonTranslateAsyncClient(credentialsProvider)
}
{% endhighlight %}

The code creates a client object (which wraps the actual HTTP-based API) and links a credentials provider so that our unauthenticated IAM user credentials are used for the request.

## Translate some text

We next need a function to translate the text. Even with the setup we used to get here, it’s remarkably simple to use:

{% highlight kotlin %}
/**
  * Translate some text, returning it in a callback
  */
private fun translate(s: String, callback: (String) -> Unit) {
    Log.d(TAG, "translate($s)")
    val translateRequest = TranslateTextRequest().apply {
        sourceLanguageCode = "en"
        targetLanguageCode = "es"
        text = s
    }

    translateClient?.translateTextAsync(translateRequest, object : AsyncHandler<TranslateTextRequest, TranslateTextResult> {
        override fun onSuccess(request: TranslateTextRequest?, result: TranslateTextResult?) {
            Log.d(TAG, "onSuccess: ${result!!.translatedText}")
            callback(result!!.translatedText)
        }

        override fun onError(exception: Exception?) {
            Log.e(TAG, "Error when translating: ${exception?.message}")
        }
    })
}
{% endhighlight %}

The `translateRequest` contains just three fields — the source and destination languages and the text to be translated. We are going over the network, so this is run asynchronously using a Future `AsyncHandler`. When the response is received, the result is the translated text.

In this case, I’ve opted to specify the language as spanish. You can use Arabic (ar), simplified Chinese (zh), French (fr), German (de), Portuguese (pt) and Spanish (es). Just change the `targetLanguageCode` accordingly. You can also add a settings panel or options list to choose the language.

The only thing left is to write the text to the prepared text view:

{% highlight kotlin %}
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)

    if (requestCode == REQUEST_SPEECH_RECOGNIZER) {
        if (resultCode == RESULT_OK) {
            data?.let {
                val results = data.getStringArrayListExtra(RecognizerIntent.EXTRA_RESULTS)
                recorded_speech.text = results[0]
                translate(results[0]) {
                    Log.d(TAG, "translated text = ${it}")
                    runOnUiThread {
                        translated_text.text = it
                    }
                }
            }
        }
    }
}
{% endhighlight %}

The main thing to remember here is that the `translateClient` runs on a background asynchronous thread. You cannot update the UI on that thread so you have to explicitly switch to the UI thread in the callback.

## Conclusion

I hope you enjoyed this foray into mobile machine learning. There are many more capabilities in the [Amazon Machine Learning suite](https://aws.amazon.com/machine-learning/), including natural language processing, text-to-speech, image recognition, and custom deep learning capabilities. AWS even has its own [speech-to-text service](https://aws.amazon.com/transcribe/) if you don’t feel like sending more data to Google.

You can check out this app on [my GitLab repository](https://gitlab.com/adrianhall/speech-translator-app).

{% include links.md %}
