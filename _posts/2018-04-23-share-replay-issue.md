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
This post describes an issue with the current `shareReplay` operator in RxJS. There has been an open issue for this on Github, but as it has been open for quite some time now. The issue might introduce some memory leaks, so I decided to write about it.

You can find the Github issue <a href="https://github.com/ReactiveX/rxjs/issues/3336" target="_blank">here</a>. There is already a PR for this issue that has been discussed <a href="https://github.com/ReactiveX/rxjs/pull/3354" target="_blank">here</a>. However, it is still unclear what the desired behavior should be. This needs to be discussed by the core team members

Hopefully this will get resolved and it will get fixed really quick :). For the time being, it made sense to warn people about it.

## Let's find out the issue

Let's take a look at the following code.

```typescript
const interval$ = interval(1000)
  .pipe(
     tap(console.log),
     mapTo('nextValue'),
     take(10),
     shareReplay(1),
  );
```

We are defining an interval stream. Let's assume that we need this interval stream to create multiple other streams. In that case, we need to transform our stream from a cold stream to a hot one by multicasting it. If we do not do this we are basically subscribing multiple times to the interval stream, and we are triggering the interval multiple times.

Using the `shareReplay` operator, we are multicasting our `interval$`. So far so good right!

Let's subscribe to this stream and stop subscribing after '3000ms'. 

```typescript
const subscription = interval$.subscribe(console.log);

setTimeout(() => {
  subscription.unsubscribe();
}, 3000);
```
`shareReplay` uses a `refCount` under the hood. This will make sure that the source stream is subscribed to when the subscribers count goes up to 1 or higher. Whenever we unsubscribe, the `refCount` of the subscription is dropping to 0. In that case, we expect that the source stream (being our interval) is cleaned up and it stops emitting values.

Let's see what happens. A live example of the code can be found here:

<a class="jsbin-embed" href="http://jsbin.com/kocagizoje/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.1.4"></script>

As you can see, the interval part of our stream keeps emitting whenever we unsubscribe. This might lead to some unexpected behavior in our code. Whenever the `interval$` has no more subscribers, it doesn't make sense to keep the source subscription active. 

### Defining the problem

Whenever we are using the `shareReplay` operator, we must be very careful. The `shareReplay` operator does not clean up streams when they have not yet completed. In our case, the `interval$` keeps emitting values after we stopped listening. This will introduce a memory leak into our application.

### Fixing the problem

Whenever the stream we are multicasting will have multiple values, or even don't complete on their own, it might be better to use the combination `publishReplay(1).refCount()` for the time being. 

