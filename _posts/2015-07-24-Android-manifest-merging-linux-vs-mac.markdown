---
layout: post
title:  "Android Manifest merging: A cross-platform war story"
date:   2015-07-24 00:08
categories: android programming tooling gradle
author: Sven Bendel
---
This is the story of a wannabe dev-op trying to find his way in the jungle of Android Manifest merging via the Android gradle plugin.

Due to the various distribution channels of our app and the different permission configurations being necessary for those different stores (especially Amazon and the Google Play Store differed in our case) we recently decided to implement those manifest differences via the rather convenient way of mirroring the relevant parts of the manifest via flavor source folders. The Android gradle plugin makes this [really easy][multi-flavor-documentation]. Look at the following exemplary gradle Android project structure:

~~~~
app
 |_...
 |_src
 |  |_forAmazon <- spoiler: notice the LOWER-case letter at the beginning!
 |  |  |_AndroidManifest.xml
 |  |_forGoogle
 |  |  |_AndroidManifest.xml
 |  |_forSamsung
 |  |  |_AndroidManifest.xml
 |  |_free
 |  |  |_java
 |  |  |_res
 |  |  |_...
 |  |  |_AndroidManifest.xml
 |  |_paid
 |  |  |_java
 |  |  |_res
 |  |  |_...
 |  |  |_AndroidManifest.xml
 |  |_main
 |  |  |_java
 |  |  |_res
 |  |  |_...
 |  |  |_AndroidManifest.xml
 |  |_test
 |  |  |_...
 |  |  |_AndroidManifest.xml
 |_...
~~~~

 The flavors are defined in the ´android´ directive of the app's ´build.gradle´ like so:

 {% highlight groovy %}
 android{
   flavorDimensions "flavor","store"
   productFlavors {
     free {
       dimension "flavor"
     }
     paid {
       dimension "flavor"
     }
     forAmazon {
       dimension "store"
     }
     forSamsung {
       dimension "store"
     }
     forGoogle {
       dimension "store"
     }
   }
 }
 {% endhighlight %}

 Fairly straight forward, huh? Executing a task like ´:app:assembleFreeForGoogle´ merges the manifests of the _free_ and the _forGoogle_ with the _main_ Manifest.

 It was short before our next release when disaster struck. We're using a Linux distribution on our CI server to execute tests and build nightly, RC and release builds within [docker][docker] containers to ensure repeatable builds. During testing everything was fine. No one checked whether there were any issues with the permissions cause they were not that vital when using the app (a rather spectacular oversight in the first place!) - the permissions mainly covered tracking stuff. However when uploading the updated APK to the Google Play Store we were confronted with a warning that we had declared less permissions than in the previous version of our app. Major bummer!

 So we examined the APK's manifest file and - as usual - Google was right: the permissions declared in our store flavor manifests were simply missing. But how could that be? Together with our dev-ops we tried to rebuild the application manually on our local machines and everything was okay. However no matter what we did, the CI always omitted the permissions when building the APK. So we checked for all sorts of configuration files, tool versions and various other desperate things including googling for all sorts of increasingly complex search terms, but to no avail. Then we pulled the docker container to our local systems and ran the build via docker. And guess what? Everything was fine. To put it in a nutshell: running the builds with the exact same configuration (ensured by the docker container) on a Mac went perfectly well while running the build on our Linux CI caused manifest merging issues. So there had to be some difference with how OS X and Linux handled the build.

 So far so bad. We examined the flavor folders more closely and finally found something weird: the store flavor folders all started with a capital letter while the android convention for specifying flavor folders encourages using lower-case letters. However on a Mac this doesn't make any difference as its file system is case insensitive - in opposite to a Linux file system where the folders simply weren't picked up by the manifest merger who looked for folders starting with a lower-case letter!

 The fix was easy. **We changed the flavor folder names to start with a lower case letter and ran the CI builds again - huge success!** All permissions were included into the Android manifest.

 Finding the issue behind this seemingly weird behavior took 2 days. It was one of the craziest tooling problems I've ever encountered caused by a very simple mistake. One could argue against _convention over configuration_ now, but I'd say: RTFM! Do it once, do it twice and then check again your implementation! This may sound harsh, but it's the price we should be glad to pay for fewer and shorter configuration files and setup time.

 So status green now? Not quite. There's still one thing I don't understand and which worries me a bit although everything is running fine now. Why didn't the docker container which was configured to use a Debian Linux as OS fail to complete the manifest merging when run on a Mac via boot2docker? Maybe someone stumbles over this post in the future who can answer this question. I'm really curious to sort this last thing out.

[multi-flavor-documentation]: http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Multi-flavor-variants
[docker]: https://www.docker.com/
