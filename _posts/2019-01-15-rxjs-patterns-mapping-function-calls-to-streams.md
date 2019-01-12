---
layout: post
title: RxJS patterns - Mapping function calls to streams
published: true
description: RxJS patterns mapping function calls to streams
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

// insert previous section

Now, I want to cover how you map function calls to streams. Sometimes we need to transform a function call into a stream. Let's say that we are integrating with an external library. We can pass a function to this library and every time this function gets called, we need to perform an action. 

For example, we have a date picker to which we can pass a function `dateChange` that gets called when the date is updated. Whenever our date changes, we want to perform a backend call to fetch some data and show it onto the screen. 

In RxJS code, I would write something like this:

```typescript
const dateChange$ = ...

const data$ = dateChange$.pipe(
	switchMap(date => this.service.getData(date),
);
```

In this scenario, I'm defining `data$` in terms of what it depends on, being the date. Because we define it like this and RxJS is a push based paradigm, every time the date changes, the `data$` will be updated and get a new value.

But the problem here is, how do we get a `dateChange$`. We are integrating with a third party library that is going to call a function every time the date changes, it does not give us a stream. And we need a stream to write the reactive code we had in the snippet above.

## Transforming a function into an Observable

An `Observable` is something that encapsulates 'work' (to understand what I mean with 'work' please read the first pattern I covered INSERT LINK). The encapsulation here means that we cannot define what that 'work' is. Once we get an `Observable` we can no longer change that 'work'. 

However, there is a building block in RxJS were we can decide whatever an `Observable` produces and this is called a `Subject`!

A complicated definition of a `Subject` is that it is both an `Observable` and an `Observer`. I like to think of it as being an `Observable` that has no work encapsulated and where you can produce the values. So you can send the next, error and/or complete event(s).

Let's see a code example on how we can leverage `Subject`s to transform a function call into an `Observable`.

```typescript
const dateChange$ = new Subject();

someLibrary.onDateChange(
	(date) => dateChange$.next(date)
);
```

And that's it! We pass a function to the third party library that only emits the value it gets onto the created `dateChange$` `Subject`. And because a `Subject` is also an `Observable`, we can just plug this into the snippet we had above.

```typescript
const dateChange$ = new Subject();

someLibrary.onDateChange(
	(date) => dateChange$.next(date)
);

const data$ = dateChange$.pipe(
	switchMap(date => this.service.getData(date),
);
```
Tada!

## When to use this

Some concrete examples where to use this are:

- When a function gets called by a third party library 
- To transform `@Output`s in Angular into streams













