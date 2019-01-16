---
layout: post
title: RxJS patterns - restarting work
published: true
description: RxJS patterns how to restart an observable
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover2.jpg'
---

Having used RxJS for a while now, I've started to see patterns that I'm using over and over again. In this blogpost, or better, series of blogposts, I want to share those patterns that I'm frequently using so that you can apply them in your own code.

I'll do this by describing them as high level as possible, but still provide you with some examples on where they can be applied.

## What is 'work'?

When conducting interviews and discussing RxJS, I tend to ask people how they would describe an `Observable` in only a couple of sentences. I'm well aware that this is difficult and quite hard to describe. Here is my personal attempt:

> An `Observable` is a wrapper around some work. That work can be triggered by subscribing in response to which one or multiple results can be pushed towards us. This might happen both synchronously and asynchronously. 

The reason I'm mentioning this is because it will help me in explaining what I mean with the term 'work' in the context of an `Observable`. Whatever is being triggered by subscribing to an `Observable` is something that I will refer to as 'work'.

## Why do we need to restart work?

Let's say that we have an `Observable` whose 'work' is actually triggering a backend call. We might want to execute this backend call multiple times, hence we want the 'work' that this specific `Observable` does to be executed multiple times.

Thanks to the fact that `Observables` are cold by default, we can accomplish this by subscribing to this `Observable` multiple times. Every subscription will trigger a backend call.

## How can we do this?

### Non reactive

A non-reactive solution to do so could be wrapping the subscription in a function and calling it over and over again every time we wish to subscribe.

```typescript
const executeBackendCall = () => {
	this.someObs$.subscribe((result) => {
	   // do something here
	}
}
```

But this obviously has some downsides to it. Aside from the fact that is not reactive, we have a subscription to manage for every function call, we can no longer chain this with other operators (as we manually subscribe), and so on.

### Reactive

We want to implement this in a more reactive way. We can do this by no longer thinking in terms of function calls but in term of a series of triggers where we want to do something. We need a stream that is triggered every time we want a certain action to occur.

Let's assume we have such a stream. Whenever that stream fires, we want to do some 'work'. Remember that 'work' is something that is encapsulated in an `Observable`.

Translated to code this means that we want to map a next event to an `Observable`.


```typescript
const work$: Observable<T> = ...
const trigger$ = ...

const workExecutedOnTrigger$: Observable<Observable<T>> = trigger$.pipe(
	map(triggerValue => work$)
);
```

In this piece of code we have an `Observable` called `work$`, containing the 'work' we want to do when the `trigger$` fires. 
We can create a new stream called `workExecutedOnTrigger$` that maps the trigger event to the `work$`.

The problem with this code is that the result of this action is an `Observable<Observable<T>>` (also called a higher order observable) whilst we would actually like to have an `Observable<T>`.

To accomplish this, we can use a flattening operator. We could use `switchMap` for example.

```typescript
const work$ = ...
const trigger$ = ...

const workExecutedOnTrigger$: Observable<T> = trigger$.pipe(
	switchMap(triggerValue => work$)
);
```

And this is it! Now we have an observable `workExecutedOnTrigger$` that will execute our `work$` and will restart that every time our `trigger$` fires. It is restarted as the `switchMap` operator will do the following:

1. the `trigger$` fires a next event
2. the `switchMap` operator takes that event and maps it to `work$`
3. the `switchMap` operator subscribes to that stream
4. Every event emitted by the `work$` is being passed down
5. if the `trigger$` fires a new event, the previous execution of `work$` is unsubscribed from
6. Step 2 through 5 is repeated

Thanks to the `switchMap` operator, we can restart the 'inner' `work$`. This means that we can **restart our work**. 

**Note:** To create the `trigger$` we can use a `Subject`. I'll cover this in the next pattern 'mapping function calls to streams'.

## When to use it

As promised, here are some concrete example when you would use this:

- In an autocomplete, we want to execute a backend call every time the user types something new. We want to restart the call to the backend.
- We fetch some data when the page loads for the first time. When we want to do this a second time (after data has been updated for example), we want to re-execute that call.

```typescript
this.data$ = this.load$.pipe(
   switchMap(_ => this.service.loadData())
);
```
In this example `load$` will emit at start up and every time the data needs to be loaded. This is a very common scenario where this concept of restarting work can be used.

 ## Special thanks
 
 Special thanks to <a href="https://twitter.com/elmd_" target="blank">Dominic Elm</a> for reviewing!
















