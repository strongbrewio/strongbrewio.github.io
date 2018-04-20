---
title: Be careful when using shareReplay
published: true
author: kwintenp
description: Describes a bug with the current implementation of shareReplay
layout: post
navigation: True
date:   2018-04-23
tags: RxJS
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover8.jpg'
---
This post describes an issue with the current `shareReplay` operator in RxJS. There has been an open issue for this on github but as it has been open for quite some time now. The issue might introduce some memory leaks so I decided to write about it. You can find the github issue <a href="" target="_blank">here</a>. Hopefully it will get fixed really quick :).

## Let's find out the issue

Let's take a look at the following code.

```typescript
const interval$ = Rx.Observable.interval(1000)
  .do(console.log)
  .mapTo('nextValue')
  .shareReplay(1);
```

We are defining a
