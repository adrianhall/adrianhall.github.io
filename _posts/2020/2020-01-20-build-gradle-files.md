---
title: "Adding a pre-build step to Android Studio builds"
categories:
  - Android
---

I'm currently writing a cloud-native app with an Android (and iOS) front end.  I've got my backend configuration written using [Terraform](https://terraform.io), and it outputs a file called `infrastructure.json` that describes the backend in a JSON format.  Now, the question becomes "how do I get that `infrastructure.json` file into my front end code?"  

A simple version of this would be to ensure I copy the file every single time I update it.  However, a better solution would be to ensure the file exists and to copy it to the right place.  An even better solution would be to pull the information I need and write out the correct JSON format for Android Studio.

In this example, I'm going to read in a JSON file from outside the Android Studio area, and write a JSON file to `res/raw`.

## Gradle is Groovy

Literally.  The language underlying gradle, which is used for builds by Android Studio, is [Groovy](http://www.groovy-lang.org/).  This is good for us, because it means we have a lot of language features at our disposal.  I'm going to use the file management and JSON serialization features in this example.

Start by creating a file called `prebuild.gradle` at the same level as the project `build.gradle`.  Fill it with these contents:

```gradle
import groovy.json.JsonSlurper
import groovy.json.JsonOutput

task generateAppConfigurationFile(type:Exec) {
    def jsonSlurper = new JsonSlurper()
    def data = jsonSlurper.parse(new File('../../infrastructure.json'))

    // If necessary, construct a new JSON object, then continue
    // Change data here to the new object
    def json_str = JsonOutput.toJson(data)

    // Write the json_str to the configuration file
    def config_file = new File('app/src/main/res/raw/config.json')
    config_file.write(JsonOutput.prettyPrint(json_str))
}

build.dependsOn generateAppConfigurationFile
```

This creates a task called `generateAppconfigurationFile`.  It reads in the JSON file specified (relative to **this gradle file**).  Once read in, you could create a new object.  For instance:

```gradle
def newData = [:]
newData.baseUrl = data.base_url.value
newData.clientId = data.client_id.value
```

Once I've got the right data, I create the JSON string, and finally write it to the configuration file (again, relative to **this gradle file**). 

The final line is the important one - it hooks this task into the build process, ensuring the build phase depends on this task, thus ensuring that the configuration file will exist before building starts.

If `infrastructure.json` or the `app/src/main/res/raw` directory doesn't exist, then the task will fail, and hence the whole build will fail.  I've placed a `.gitignore` file in the `app/src/main/res/raw` directory to explicitly prevent `config.json` from being checked in.  This will also ensure that the directory exists when you clone the repository.

## Hooking the prebuild step into the configuration

Now, go to the module-level `build.gradle` file.  For me, this is `app/build.gradle`.  Add the following line to the top:

```gradle
apply from: '../prebuild.gradle'
```

Mine is right under the `apply plugin:` lines.

## Build away!

Whenever you build your app, the configuration file will be re-generated for you!
