---
title: How to test a custom Android view with Robolectric
tags: Kotlin Android Testing Robolectric
key: android-custom-view-testing

cover: /assets/images/robolectric.png
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /assets/images/robolectric.png
---

Working at an enterprise that needs a custom component library with each component being its own snow flake and having more complicated logic than the next. I found myself needing a way to test the logic in these views to ensure I could iterate quickly and not break anything in the process. 

<!--more-->

Having a lot of prior experience unit testing business logic at the presentation and domain layer, it was always best practice to remove all Android references as this can be very tricky to mock. So testing the logic in an Android view was somewhat foreign to me, until I came across a [medium article](https://proandroiddev.com/testing-custom-views-with-robolectric-4-bac7226f4a52) that had a very simple example of how this could be achieved with [Robolectric](http://robolectric.org) by asserting the text in `TextView`. So in this post I will try to expand on this concept with a slightly more complex example and give some tips for utilizing Kotlin to write more idiomatic tests. 

At the time this post was written, [Jetpack Compose](https://developer.android.com/jetpack/compose) was only in the alpha stage which provides a more robust solution for building and [testing](https://developer.android.com/jetpack/compose/testing) custom components. Enterprises like stable things though, so that's what this post will describe and encourage you to check out compose if it is stable or you want to be on the bleeding edge of technology. 
{:.warning}

## Create a Components Library

If you were wanting to build a component library, you can create a new module under `File -> New -> New Module...` and select Android library. 

## Create a Custom Android View 

The component that will be built is a material card which has a lock button, text description, and a background ripple that is only active the card is unlocked. 

![](/assets/images/android_custom_view.gif)


## Source code

[Github Repository](https://github.com/plusmobileapps/test-custom-android-view)

## Resources 

* [Create custom Android view](https://developer.android.com/training/custom-views/create-view)
* [Robolectric documentation](http://robolectric.org)