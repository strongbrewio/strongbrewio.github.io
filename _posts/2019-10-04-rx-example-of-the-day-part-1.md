---
layout: post
title: RxJS examples of the day (part 1)
published: true
description:  A couple of examples practical examples on RxJS
author: kwintenp
comments: true
date: 2019-10-04
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover5.jpg'
---

Having worked with and teaching RxJS for quite some time now, I always felt that the way RxJS is explained is very theoretical. For that reason, I started tweeting out these practical RxJS examples that I basically plucked from my own codebases. 
This blog post will hopefully be part of a series (depending on my inspiration ☺️). Here are the first 5.

### Handle replaying of keydown events

The browser replays 'keydown' events for as long as the user keeps pressing the button. Often times, you just want to have the 'keydown' event as a trigger but want to remove the duplicate ones without constantly removing and reading the eventlistener.

<div style="display: flex; justify-content:center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/RxExampleOfTheDay?src=hash&amp;ref_src=twsrc%5Etfw">#RxExampleOfTheDay</a> The browser replays &#39;keydown&#39; events for as long as the user keeps pressing the button. <br><br>This is a stream that emits &#39;keydown&#39; or &#39;keyup&#39; whenever the user presses/releases a button. Without having duplicate events 🥳<a href="https://twitter.com/stackblitz?ref_src=twsrc%5Etfw">@stackblitz</a>: <a href="https://t.co/HrJQ3AAEFi">https://t.co/HrJQ3AAEFi</a> <a href="https://t.co/t4jlu1WmnO">pic.twitter.com/t4jlu1WmnO</a></p>&mdash; Kwinten Pisman (@KwintenP) <a href="https://twitter.com/KwintenP/status/1160841804603973632?ref_src=twsrc%5Etfw">August 12, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

### Using the conditional `iif` operator

The `iif` operator helps when your initial stream depends on a condition. Without the operator, we would have to write a pretty ugly (and imperative) 'if-else'.

<div style="display: flex; justify-content:center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/RxExampleOfTheDay?src=hash&amp;ref_src=twsrc%5Etfw">#RxExampleOfTheDay</a> Whenever your initial stream depends on a condition, it&#39;s best to use the `iif` operator instead of declaring a variable and then assigning it in an if-else. <a href="https://twitter.com/hashtag/rxjs?src=hash&amp;ref_src=twsrc%5Etfw">#rxjs</a> <a href="https://t.co/XePmj7ZrhM">pic.twitter.com/XePmj7ZrhM</a></p>&mdash; Kwinten Pisman (@KwintenP) <a href="https://twitter.com/KwintenP/status/1162090144410849280?ref_src=twsrc%5Etfw">August 15, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

### When and how many times is a stream subscribed to

Knowing when and how many times a stream is subscribed to can be extremely valuable information to have when working with RxJS. If you for instance have multiple backend calls that are triggered, this info can help pinpoint the issue. 

<div style="display: flex; justify-content:center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/RxExampleOfTheDay?src=hash&amp;ref_src=twsrc%5Etfw">#RxExampleOfTheDay</a> A small utility to make debugging a little easier. An operator that logs out IF an Observable was subscribed to AND how many times it was subscribed to.<br>Has saved me quite some time debugging <a href="https://twitter.com/hashtag/rxjs?src=hash&amp;ref_src=twsrc%5Etfw">#rxjs</a> 😅<br><br>Stackblitz: <a href="https://t.co/03bxrUg53H">https://t.co/03bxrUg53H</a> <a href="https://t.co/8EpKZNrxXg">pic.twitter.com/8EpKZNrxXg</a></p>&mdash; Kwinten Pisman (@KwintenP) <a href="https://twitter.com/KwintenP/status/1162382060687974400?ref_src=twsrc%5Etfw">August 16, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

### Optimising your autocompletes

When the user starts typing, you typically want to remove the existing results in an autocomplete as they don't match what the user was typing no more. You can do this by branching existing streams instead of resorting to side effects or `Subjects`.

<div style="display: flex; justify-content:center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr">#<a href="https://twitter.com/hashtag/RxExampleOfTheDay?src=hash&amp;ref_src=twsrc%5Etfw">#RxExampleOfTheDay</a> With an autocomplete, clear the old results when the user types. Don&#39;t use Subjects or side effects with a tap to do that! Just create multiple streams from the &#39;input$&#39; for &#39;resetting&#39; and &#39;searching&#39; and then merge them. <a href="https://twitter.com/hashtag/rxjs?src=hash&amp;ref_src=twsrc%5Etfw">#rxjs</a><br><br>SB: <a href="https://t.co/cRyHU9LF2I">https://t.co/cRyHU9LF2I</a> <a href="https://t.co/V0J1Sg6e41">pic.twitter.com/V0J1Sg6e41</a></p>&mdash; Kwinten Pisman (@KwintenP) <a href="https://twitter.com/KwintenP/status/1163452741148168192?ref_src=twsrc%5Etfw">August 19, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>

### Using exhaustMap as a flattening operator

Sometimes you want to avoid the user constantly canceling calls. If a call is already on it's way, we probably want to continue that one first. Using `exhaustMap` is the perfect operator for that.

<div style="display: flex; justify-content:center;">
<blockquote class="twitter-tweet"><p lang="en" dir="ltr"><a href="https://twitter.com/hashtag/RxExampleOfTheDay?src=hash&amp;ref_src=twsrc%5Etfw">#RxExampleOfTheDay</a> Most users, including me, are impatient and when something feels slow, we click a (refresh) button multiple times. <br>This behavior in combo with `switchMap` can be a dangerous combination! Often `exhaustMap` is a better option <a href="https://twitter.com/hashtag/rxjs?src=hash&amp;ref_src=twsrc%5Etfw">#rxjs</a><br><br>SB: <a href="https://t.co/rTKj5MpcAU">https://t.co/rTKj5MpcAU</a> <a href="https://t.co/zsxYcE0h1c">pic.twitter.com/zsxYcE0h1c</a></p>&mdash; Kwinten Pisman (@KwintenP) <a href="https://twitter.com/KwintenP/status/1164145793806360582?ref_src=twsrc%5Etfw">August 21, 2019</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
</div>