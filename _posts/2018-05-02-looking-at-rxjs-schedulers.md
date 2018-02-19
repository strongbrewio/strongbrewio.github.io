---
title: What are those schedulers in RxJS
published: true
author: kwintenp
description: A deep dive into what RxJS schedulers are
layout: post
navigation: True
date:   2018-02-05
tags: RxJS
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover3.jpg'
---

One topic in RxJS for which it is quite hard to find proper documentation/blogposts, is 'Schedulers'. 'Schedulers' are a way to control the timing strategy use to execute tasks. The main reason for this is that the authors of RxJS did a great job in abstracting this logic. They used the principle of least concurrency as the default scheduling strategy which makes sure that in most cases, we don't have to think about changing the default.

This post will examine the different scheduler and explain the differences between them. 

## Async in javascript
To understand the difference between the different schedulers, we must first examine how async works in javascript. No worries, we won't go in to deep :).
Javascript is a single threaded language that uses an event loop to handle asynchronous operations. One of my favorite talks ever covers this topic really well. It's by Philip Roberts and you can find a recording <a href="https://www.youtube.com/watch?v=8aGhZQkoFbQ" target="_blank">here</a>. Be sure to check it out you are unsure what the event loop is and how it works. 

### Understanding by example

Let's look at an example.

```typescript
console.log('script start');

setTimeout(function() {
  console.log('setTimeout');
}, 0);

Promise.resolve().then(function() {
  console.log('promise1');
}).then(function() {
  console.log('promise2');
});

console.log('script end');
```

If you run this piece of code, the result will look like this:

```typescript
script start
script end
promise1
promise2
setTimeout
```
If you know how the event loop works, you understand that the 'script start' and 'script end' were logged first. The 'setTimeout' and 'promises' that get triggered were put on a queue to be executed when the call stack cleared. 

But what's a lesser know fact is that there are different queues. There is a microtask and a macrotask queue. 

A promise gets added to the microtask queue and a setTimeout to the macrotask queue.
Whenever the call stack is cleared, the microtasks queue gets cleared first. Hence the 'promise1' gets logged. 

When a microtask is finished, the rest of the microtasks queue gets executed until it is empty. That's the reason 'promise2' is logged next. So even the fact that the second promise gets scheduled during the execution of the first, it still gets executed before the setTimeout. 

Lastly, when the microtask queue is cleared, the next task is picked from the macrotask, and the setTimeout is logged.

There you have it, this is how async works in javascript! Might be a little to quick but since it's not part of this post to explain everything in detail, I'll leave it at that. And also because Jake Archibald basically explained it in perfect detail right <a href="https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/" target="_blank">here</a>.


## Schedulers










