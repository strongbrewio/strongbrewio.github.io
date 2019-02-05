---
layout: post
title: RxJS patterns - Prematurely ending work
published: true
description: RxJS patterns prematurely ending work
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---

This post is part of a series of blogpost on different RxJS patterns that I use quite often. Here are the previous ones:

- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">Restarting work</a>
- <a href="https://blog.strongbrew.io/rxjs-patterns-mapping-function-calls-to-streams/" target="_blank">Mapping function calls to streams</a>
- <a href="https://blog.strongbrew.io/rxjs-patterns-conditionally-executing-work/" target="_blank">conditionally executing work</a>

The next pattern I want to discuss when we you want to prematurely end work. There are many scenarios when you would want to do this. We can start some work (aka an `Observable`) and want to kill that subscription if our  user navigates to a different page, the user changes something which should trigger a different call, ...

## How can we do it

One thing I really like about RxJS is how declarative it is. You can quite often translate what you want into a sentence and translate that sentence into code. 

Let's say that we want to listen to a certain stream, until a certain event happens. An extremely common scenario is when you are working with a UI framework that works with components. Often you want to: 'Listen to this specific `Observable` until the component gets destroyed'.

Translated into code, we could have something like this:

```typescript
workWeWantToDo$.pipe(
	listenTillThisEventHappens(componentDestroyed)
).subscribe();
```
**Note:** `listenTillThisEventHappens` is not an operator :).

The `listenTillThisEventHappens` can be replaced by the `takeUntil` operator which does exactly this. And we can create a stream from a lifecycle hook that gets called via our UI framework. Checkout the other pattern 'mapping function calls to streams' to see how you can accomplish this.

The resulting code is:

```typescript
workWeWantToDo$.pipe(
	takeUntil(componentDestroy$)
).subscribe();
```

When creating this `Observable` we use the `takeUntil` operator. This operator will subscribe to the stream above (if it was subscribed to itself of course) and will proxy these values until the `componentDestroy$` emits. Then the `takeUntil` will kill the subscription.

## When to use this

Everytime you are thinking: 'I want to stop this when that happens'.

- To kill subscriptions when a component is destroyed
- When trying to know when a user stopped dragging. You can have a stream that listens for 'dragDown' and take values until a 'dragUp' events happens.










