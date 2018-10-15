---
layout: post
title: Using nx-etc checkout feature
date:   2018-10-14
subclass: 'post'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: Angular
cover: 'assets/images/cover/cover6.jpg'
---

Using [Nx](https://nrwl.io/nx) we can build applications using a monorepo style of development. If your monorepo contains a lot of applications and libraries, the development process might be impacted a little.

### Problems

* Looking for files becomes somewhat annoying as you can find lots of them, even for apps or libs that you are not working one.
* Opening up 20 apps and 50 libs in your IDE will impact its performance. While you might be only working on one or two apps.

### Solution

Nx-etc has a checkout feature that uses sparse checkouts and some scripts from Nx to only checkout apps and libs you are currently working on.
You can find an overview of the lib in the video below.

<iframe src="https://player.vimeo.com/video/295018577" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/295018577">Using nx-etc checkout feature</a> from <a href="https://vimeo.com/user79085465">Kwinten Pisman</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

You can find the code on [github](https://github.com/KwintenP/nx-etc).

**Note:** Thanks to [@juristr](https://twitter.com/juristr), [@beeman](https://twitter.com/beeman_nl) and [@chaos_monster](https://twitter.com/chaos_monster) for testing and providing feedback!
