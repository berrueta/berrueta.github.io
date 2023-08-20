---
layout: post
title:  "Using a custom apex domain with Github Pages"
date:   2023-08-20 18:49:29 +1200
category: software
tags: github jekyll dns domain gandi
---
Yesterday I refreshed my website using Jekyll and Github Pages. Today I reconfigured it to be available under my `berrueta.net` domain. Previously I had instructed my DNS registrar to redirect the traffic from `berrueta.net` to `berrueta.github.io`, but I wanted to use the apex domain instead of Github's.

The [instructions][instructions-gh] from Github are precise but not particularly clear. Fortunately I found a helpful guide by Matt Bailey specifically for my DNS registrar (Gandi), with more details about how to configure the `A` and `CNAME` records:

{% gist bbbc181d5234c618e4dfe0642ad80297 %}

I also took the opportuntity to play a bit more with some Jekyll plugins to embed some types of content.

[instructions-gh]: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
