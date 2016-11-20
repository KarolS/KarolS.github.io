---
layout: post
title: "Move from Octopress to Github-side Jekyll"
date: 2016-11-20 15:00
categories: []
---

One of the main reasons I haven't posted a lot on this blog is that the whole process of building the website was quite cumbersome and required a fully configured Ruby environment. 
Thankfully, Github offers its own Jekyll installation that automatically builds the website and allows people to submit files straight from the browser.
So I decided to spend few hours and unshackle myself from the confines of my local and soon-to-be ditched Ubuntu 14.04 installation that hosted necessary tools.
The result is, as you can see, a fully working and functionally-equivalent blog, but which I can edit in a browser.
In fact, I'm writing this post in Firefox right now.

Of course, migration was not trivial. I had to make multiple changes:

<!-- more -->

* I had to remove lots of plugins written in Ruby, since Github is not going to run arbitrary Ruby code.

* Partials and layouts had to be modified to accomodate removed plugins and other differences between my old Octopress and current Jekyll. 

* I used Compass for compiling SCSS. Since Github doesn't provide Compass integration, I just run it on my own computer and then upload compiled CSS.

* The plugin I missed the most was category pages plugin, which I emulated following the guide at http://www.minddust.com/post/alternative-tags-and-categories-on-github-pages/ 

* I rewrote the category cloud based on https://superdevresources.com/tag-cloud-jekyll/ but I adapted it to use categories as defined in the previous step. I also used the same method to inject category lists in other places.

* I also fixed the Google search box, because if I make a query qith two `q` parameters and then get redirected from `google.com` to `google.com.pl`, then one of the parameters is ignored. I solved it by hardcoding the box to use `google.co.uk`.

* I took some time to fix table styling, so that tables don't look as hideous as they used to.
