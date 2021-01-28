---
layout: post
title: "Why doesn't my theme work?"
date: 2021-01-28 20:17:09 +0800
tags: android rro
---
<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/chooser.png" alt="Screenshot of app chooser in Android, 'Just once' and 'Always' button are in a teal color that is hard to read given the background">
  <figcaption>In the app chooser, those two buttons don't follow the accent theme, and the color contrast is also too low.</figcaption>
</figure>

## TL;DR

The package that contains the app chooser activity, `android`, somehow didn't load the accent RRO.

What I did in the end is convert an accent RRO to static overlay so my eyes won't burn every time I need to choose an app.

## The journey

### Layout inspector
After discovering the problem, I decided to use Layout Inspector from Android Studio to look into the layout hierachy, hoping to identify some theme resolution problems.

> But, hey, doesn't that only work on debuggable apps?

```shell
magisk resetprop ro.debuggable 1
```
Now let's open the layout inspector...

<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspector-process-system.png" alt="Layout inspector, target process selector, with system:ui highlighted">
</figure>


<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspector-empty.png" alt="Layout inspector, not connected, showing nothing">
  <figcaption>Hm, why isn't it loading.</figcaption>
</figure>

And then I gave up using it.

### Layout inspector, part 2
Android Studio didn't show any error. Probably some bugs?
Since the studio is open-source, I decided to poke around the API.
<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspector-bazel-build.png" alt="Android Code Search, showing the BUILD file of layoutinspector module">
  <figcaption>Oof, Bazel. Does this mean I have to download the whole repo?</figcaption>
</figure>

<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspector-bridge.png" alt="Intellij IDEA, showing LayoutInspectorBridge.kt from layoutinspector module">
</figure>

<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspector-view-node-parser.png" alt="Intellij IDEA, showing ViewNodeParser.kt from layoutinspector module">
  <figcaption>They even provide a nice API.</figcaption>
</figure>

After manually copying dependencies and specifying some external artifacts in build.gradle, it finally compiled!

<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/inspectorlib-example-code.png" alt="Intellij IDEA, showing a code snippet that utilize layoutinspector to dump view property for a button">
  <figcaption>Some ugly code I wrote.</figcaption>
</figure>

What infomation about the view can we get?
```
parsing

meta:__name__=android.widget.Button
meta:__hash__=189189865
id=id/button_once
...
drawing:overlappingRendering=true
drawing:outlineAmbientShadowColor=-16777216
drawing:outlineSpotShadowColor=-16777216
focus:hasFocus=false
focus:isFocused=false
focus:focusable=1
focus:isFocusable=true
focus:isFocusableInTouchMode=false
misc:clickable=true
misc:pressed=false
...
misc:enabled=true
misc:soundEffectsEnabled=true
misc:hapticFeedbackEnabled=true
theme={3=android.content.res.Resources$Theme, 4=11943745, 85=forced, 86=forced, 107=forced}
meta:__attrCount__=0
...
```
Not very useful. I didn't even get the theme resource ID.

### Debugger
Fine. Can we use the debugger to check the resources resolving behavior? It turns out we can!

Following the steps in [this post](https://medium.com/@ghxst.dev/static-analysis-and-debugging-on-android-using-smalidea-jdwp-and-adb-b073e6b9ae48), I attached the IDEA debugger on the system:ui process.

```java
Resource res = this.getResources();
int color = res.getColor(res.getIdentifier("accent_device_default_light", "color", "android");
String.format("#%06X", (0xFFFFFF & color));
```
```
-> #167C80 // the default teal color in AOSP
```
Wait, the overlay isn't applied?
```java
this.getResources().getAsset();
```
<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/debugger-apkassets.png" alt="Intellij IDEA debugger, showing result of the preceding code block. In the result, 10 items are presented in mApkAssets field, none of them are accent overlay">
  <figcaption>The accent overlay is not present in the framework</figcaption>
</figure>

Quickly made an app that probes this class field.
<figure>
  <img src="{{ site.baseurl }}/assets/framework-accent-rro/user-app-apkassets.png" alt="Screenshot of an app I wrote to dump mApkAssets">
  <figcaption>The accent overlay is present in user apps</figcaption>
</figure>

Somehow the overlay didn't get loaded in the framework package, probably because it runs before user-specified accent RROs are loaded.
