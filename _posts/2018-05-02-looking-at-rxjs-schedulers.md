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

One topic in RxJS for which it is quite hard to find proper documentation/blogposts, is 'Schedulers'. 'Schedulers' are a way to control the timing strategy used to execute tasks. The main reason for this is that the authors of RxJS did a great job in abstracting this logic. They used the principle of least concurrency as the default scheduling strategy which makes sure that in most cases, we don't have to think about changing the default.

This post will examine the different schedulers and explain the differences between them. 

## Async in javascript
To understand the difference between the different schedulers, we must first examine how async works in javascript. No worries, we won't go in to deep :).

Javascript is a single threaded language that uses an event loop to handle asynchronous operations. One of my favorite talks ever covers this topic really well. It's by Philip Roberts and you can find a recording <a href="https://www.youtube.com/watch?v=8aGhZQkoFbQ" target="_blank">here</a>. Be sure to check it out if you are unsure what the event loop is and how it works. 

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
If you know how the event loop works, you understand that 'script start' and 'script end' were logged first. The 'setTimeout' and 'promises' that get triggered were put on a queue to be executed when the call stack cleared. 

But what's a lesser know fact is that there are different queues. There is a microtask and a macrotask queue. A promise gets added to the microtask queue and a setTimeout to the macrotask queue.
Whenever the call stack is cleared, the microtasks queue gets cleared first. Hence the 'promise1' is the next log statement we see. 

When a microtask is finished, the rest of the microtasks queue gets executed until the microtask queue is empty. That's the reason 'promise2' is logged next. So even if the second promise gets scheduled during the execution of the first, it still gets executed before the setTimeout. 

Lastly, when the microtask queue is cleared, the next task is picked from the macrotask queue, and the 'setTimeout' is logged.

There you have it, this is how async works in javascript! Might be a little to quick but since it's not part of this post to explain everything in detail, I'll leave it at that. And also because Jake Archibald basically explained it in perfect detail right <a href="https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/" target="_blank">here</a>.


## Schedulers

In RxJS there are different schedulers that all schedule work using a different (a)sync technique behind the scenes. Let's take a look at an example to learn a few of them. There are three statement logged on different schedulers.

```typescript
const asyncScheduler = Rx.Observable.of('')
  .startWith('async', Rx.Scheduler.async);

const asapScheduler = Rx.Observable.of('')
  .startWith('asap', Rx.Scheduler.asap);

const queueScheduler = Rx.Observable.of('')
  .startWith('queue', Rx.Scheduler.queue);


Rx.Observable.merge(
    asyncScheduler,
    asapScheduler,
    queueScheduler)
  .filter(x => !!x)
  .subscribe(console.log);

console.log('after subscription')
```

The results of running this code is:

```typescript
queue
after subscription
asap
async
```

As you can see, the 'queue' is the only statement logged synchronously. We can conclude this because it is the only statement logged before the 'after subscription' statement is logged. 
Next we can see that the 'asap' is logged before the 'async'. That is because the `asap` scheduler uses the micro task queue and the `async` uses the macrotask queue.

Now, to list all of them, we can look at the table below. In the example I didn't cover the 'animationFrame' and the 'virtualTime' scheduler. The 'animationFrame' scheduler allows you to schedule work to be executed when there is a repaint. The 'virtualTime' scheduler is something you can use to test your code in a synchronous fashion using marble diagrams.

| Scheduler | Approach |
| --- | --- |
| `Queue` | Executes task synchronously but waits for current task to be finished |
| `Asap` | Schedules on the micro task queue |
| `Async` | Schedules on the macro task queue |
| `AnimationFrame` | Relies on 'requestAnimationFrame' |
| `VirtualTime` | Will execute everything synchronous ordered by delay and mainly used in testing |


### Using schedulers in RxJS
Some operators, like 'startWith' in the example above, allow you to pass in an optional scheduler to influence when a certain task gets executed. Every operator that has this option, will also have a default value.

For example, the `debounceTime` operator uses the 'async' scheduler as a default as you can see below.

```typescript
export function debounceTime<T>(dueTime: number, scheduler: IScheduler = async)
```

## Conclusion
Schedulers influence the timing on which tasks get executed. You can change the default schedulers of some operators by passing in an extra scheduler argument.




