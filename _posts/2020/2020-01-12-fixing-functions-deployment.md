---
title: "Reducing the deployment size of your JavaScript Azure Functions"
categories:
  - Cloud
tags:
  - azure_functions
---

I'm doing some Azure Functions development in JavaScript at the moment, using the new `azure-functions-core-tools`.  One of the features it has is command line publication, like this:

{% highlight bash %}
$ func azure functionapp publish my-function-app
{% endhighlight %}

I have a single function right now (called ping).  Running the publish step creates a ZIP file:

{% highlight bash %}
$ npx func azure functionapp publish my-function-app               
Getting site publishing info...
Creating archive for current directory...
Uploading 187.62 MB [##---------------------------------------------------------------------]
{% endhighlight %}

That's a big file.  Wow!  It creates a ZIP file of the whole directory!!!!  My function isn't that big!

Looking at the directories, it's obvious which ones are the culprit:

{% highlight bash %}
$ du -ks */.           
462952  node_modules/.
8       ping/.
4       tools/.
$ du -ks node_modules/*/.
du -ks node_modules/*/. | sort -rn
451588  node_modules/azure-functions-core-tools/.
3600    node_modules/es5-ext/.
3088    node_modules/es-abstract/.
624     node_modules/ext/.
424     node_modules/type/.
412     node_modules/deferred/.
360     node_modules/resolve/.
156     node_modules/npm-run-all/.
128     node_modules/object-inspect/.
104     node_modules/es6-iterator/.
100     node_modules/es6-symbol/.
96      node_modules/object.assign/.
92      node_modules/timers-ext/.
88      node_modules/event-emitter/.
88      node_modules/es-to-primitive/.
80      node_modules/semver/.
# ...
{% endhighlight %}

Getting rid of `azure-functions-core-tools` from the published source would reduce the file size considerably.  Fortunately, the tooling provides `.funcignore` for precisely this problem.  Just add `node_modules/azure-functions-core-tools` to your `.funcignore` file.  Now, the publication looks like this:

{% highlight bash %}
$ npx func azure functionapp publish my-function-app
Getting site publishing info...
Creating archive for current directory...
Uploading 1.7 MB [##########################################################################]
Upload completed successfully.
{% endhighlight %}

It knocked about 4 minutes from my publication time, and reduced the file size down to under 2MB.

> If you use TypeScript, you should also add the `node_modules/typescript` package to the `.funcignore` as that is pretty big too.

Of course, this doesn't mean anything if the functions don't work at the end of it.  I'm happy to report that my ping API works just fine, so the `azure-functions-core-tools` isn't required within the publish package.  It's also given me 5-10 minutes back per publication cycle.

