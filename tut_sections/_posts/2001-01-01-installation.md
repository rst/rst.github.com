---
layout: tut_section
title: Basic Setup
summary: Downloading and installing the library and related tooling.
---
As is unfortunately common with development tooling, developing with
Positronic Net and its tool stack (sbt and the sbt android plugin)
involves downloading several pieces, and getting them in shape.
I'll start with instructions for getting things in place (tested on
Linux, should also work on Mac):

* Download and install the Android SDK, per 
  [instructions](http://developer.android.com/sdk/index.html).

  Note that in addition to the API "starter package" itself, you
  will also need to use the `android` tool to download the API
  and platform versions you want to use.  (The library and sample
  apps are built against version 7, also known as Android 2.1 and
  Eclair, which captures most of the current installed base.)

  However, Eclipse and the Eclipse plugin are not required.

* Set the `ANDROID_SDK_HOME` environment variable to point to the
  location where you put the SDK.  (You may wish to put this in your
  `.bashrc`, or some similar file, so it automatically happens on
  login.)
  {% highlight sh %}
    ~/src$ export ANDROID_SDK_HOME=/home/.../android-sdk-...
  {% endhighlight %}

* Download and install `sbt` version 0.11.1 or better, per
  [instructions](https://github.com/harrah/xsbt/wiki/Setup).

* Download and install the Positronic Net library itself:
  {% highlight sh %}
    ~/src$ git clone https://github.com/rst/positronic_net.git
    ~/src$ cd positronic_net
    ~/src/positronic_net$ sbt publish-local
  {% endhighlight %}

If you're downloading one of the sample apps, or trying to build
one of your own, that's enough to get started.  If you're going
through the tutorial, though, you'll probably want its sample code
as well, you'll probably want to download its sample code:

  {% highlight sh %}
    ~/src$ git clone https://github.com/rst/positronic_tutorial_todo.git
    ~/src$ cd positronic_tutorial_todo
    ~/src/positronic_tutorial_todo$ sbt android:package-debug
  {% endhighlight %}

  will build a package in the `positronic_tutorial_todo/target` directory,
  and if the emulator is already running, the following (oddly named)
  incantation will load and start the app:

  {% highlight sh %}
    $ sbt android:start-emulator
  {% endhighlight %}

  Or, of course, you can install it with `adb install` as per usual.

  (Note that the tutorial walks you through the development of multiple
  versions.  The version you get above is the one from the tutorial front
  page; all versions are available as the branches named `phase1`, `phase2`,
  `phase3` and so forth.)

And, if you're proceeding with the tutorial, you probably want to follow
the link below to get started:
