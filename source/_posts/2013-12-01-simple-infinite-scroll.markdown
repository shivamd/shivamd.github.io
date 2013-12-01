---
layout: post
title: "Simple Infinite Scroll"
date: 2013-12-01 18:23
comments: true
categories: ["code", "javascript"]
---

A lot of websites use infinite scroll including Facebook&Twitter. There are a lot of Ruby Gems that help in implementing this but it's rather simple to create your own with Javascript and avoid adding another gem to your app.

Let's say we have a ```ul``` with a id of ```videos```, inside this there are a lot of ```li``` tags that contain title's of videos.

{% codeblock lang:html %}
<ul id="videos">
  <li class="video">YouTube Video 1</li>
  <li class="video">YouTube Video 2</li>
  <li class="video">YouTube Video 3</li>
  <li class="video">YouTube Video 4</li>
  <li class="video">YouTube Video 5</li>
  <!-- more videos -->
</ul>
{% endcodeblock %}

We want to detect when a user hit's the bottom of these videos and make a call to grab more videos.

{% codeblock lang:coffeescript %}

checkVideoScroll = (e) ->
  element = $(e.currentTarget())
  if element[0].scrollHeight - element.scrollTop() == element.outerHeight()
    #make call to get more videos
$ ->
  $(".videos").on("scroll", checkVideoScroll)
{% endcodeblock %}

When the document loads we set a scroll listener on the list of videos. When a user scrolls down, we call the ```checkVideoScroll``` method. On the first line the method assigns element to be the current target, which in this case is the ```ul```.
The ```scrollHeight``` is the height of the element including content not visible on the screen. The ```scrollTop()``` gets the scroll top offset of the element. The ```outerHeight()``` is a jQuery method that returns the visible portion of the scrollable element.
{% img center /images/scroll.jpg %}

As you can see once the user scrolls all the way to the bottom, ```scrollHeight - scrollTop()``` will equal the ```outerHeight()``` and that's when we make a call to get more videos. 

As you can see infinite scroll is simple with jQuery.
