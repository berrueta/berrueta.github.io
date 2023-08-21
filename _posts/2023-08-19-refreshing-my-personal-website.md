---
layout: post
title:  "Refreshing my personal website with Github Pages and Jekyll"
date:   2023-08-19 15:49:29 +1200
categories: software
tags: jekyll github
---
I created my first personal website around 1996 in the now defunct [Geocities][geocities]. Later I maintained a personal website in [asturlinux.org][asturlinux], the local Linux user group that I co-founded. Then I moved to just blogging in blogpost.com and my personal website was no longer maintained. After many years of neglect, today I spent a bit of time setting up a new personal website using Github Pages.

There is not much to be seen here for now. I may eventually add some content. For now here are some links:

* An archived version of the [abandoned website][old2007] from asturlinux.org (in Spanish), which has not been updated since 2007.
* The [blog][alotroladodelpajares] that I wrote during my decade living in Sydney, Australia, between 2012 and 2022.

This new website is hosted in Github Pages. A few technical details: the website is completely static. Setting it up using the instructions from Github was very easy. I'm using VSCode to edit the contents because I want to get a bit more familiar with that IDE (I mostly use Jetbrains IDEA in my daily job). I installed a few VSCode plugins for Jenkins and Github. The only hiccups so far were:

* I was experiencing some errors when running `bundle install`. The problem was that the version of the `bundle` gem was a bit oldie. I ran `gem install bundler` again followed by `bundle --full-index` and that solved the problem.

* The Jekyll plugin for VSCode was failing to start the Jekyll server, but the error message was not obvious. After a bit of Googling, I discovered that it aparently it needs an additional dependency. I ran `bundle add webrick` and the problem disappeared.

[old2007]: /before2007/
[alotroladodelpajares]: https://alotroladodelpajares.blogspot.com/
[asturlinux]: https://web.archive.org/web/20060101024504/http://www.asturlinux.org/
[geocities]: https://en.wikipedia.org/wiki/GeoCities
