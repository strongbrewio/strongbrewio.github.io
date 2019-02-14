---
layout: post
title: Why we have a scan for Observables but not for Arrays
published: true
description: Why we have a scan operator for Observables but not for Arrays
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover3.jpg'
---

A while ago, during a training, I was explaining the `scan` operator in RxJS and how you can use it to accumulate a 'calculated' value over time. At some point one of the participants asked why there isn't a `scan` operator for `Arrays`? They both have a `reduce` method, so why no `scan` right?

And actually it's a really interesting question which showed he didn't understand important differences between `Arrays` and `Observable`s.

## What does `scan` do in RxJS

Let's take a look at the definition of the scan operator (from the <a href="http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#instance-method-scan" target="_blank">ReactiveX.io</a> site):

> Applies an accumulator function over the source `Observable, and returns each intermediate result, with an optional seed value.

If you don't know what the `scan` operator does, this definition might not make that much sense. Let's first look at an example on how we can use the operator and then revisit this.

Let's say that we have an `Observable<number>` and we want to visualize the sum of all of the numbers on the screen. This is something we can use the `scan` operator for.  

```typescript
const numbers$ = Observable.create(observer => {
  observer.next(1);
  setTimeout(() => observer.next(2), 100);
  setTimeout(() => observer.next(3), 200);
  setTimeout(() => observer.next(4), 400);
  setTimeout(() => observer.next(5), 300);
  setTimeout(() => observer.next(6), 500);
})

numbers$.pipe(
  scan(
  	(acc, cur) => {
    	return acc + cur;
  	}, 
  	0 // <-- initial value
  )
).subscribe(console.log);
```

The `scan` operator allows us to calculate the value by passing in an accumulator function. This function will get called with the 'previous' or initial value and the 'current' value and allows us to calculate the sum. 

Let's type out the values that will go into the accumulator function.

On the first run, the initial value will be used (second parameter passed to the `scan` operator) and the first value from the `Observable`. The calculated value will be 1 (as per our function implementation). 

```typescript
// acc: 0, cur: 1 -> 1
```

Next time the function gets called, the `acc` is going to be the previously returned value and the `cur` is the next value from the `Observable`. This process repeats for every value emitted by the `Observable`. Below you can see all of the other inputs and outputs of the accumulator function.

```typescript
// acc: 1, 		cur: 2 -> 3
// acc: 3, 		cur: 3 -> 6
// acc: 6, 		cur: 4 -> 10
// acc: 10, 	cur: 5 -> 15
// acc: 15, 	cur: 6 -> 21
```

You can find the StackBlitz example below (open the console to see the result):

<iframe style="width: 100%; height: 400px" src="https://stackblitz.com/edit/rxjs-ylayki?embed=1&file=index.ts"></iframe>

**Note:** You might notice that calculating values over time is something that can be really useful to calculate state. My friend <a href="https://twitter.com/juristr" target="_blank">Juri Strumpflohner</a> has written an interesting <a href="https://juristr.com/blog/2018/10/simple-state-management-with-scan/" target="_blank">article</a> about this very topic.

## Will you please tell us why there is no Array.scan

Now that we know what the `scan` operator for RxJS does, let's revisit the definition:

> Applies an accumulator function over the source `Observable`, and returns each intermediate result, with an optional seed value.

We already know what the accumulator function is. We also get what the seed value is (our initial value, in our example 0). But I want to focus on the other part of the definition, 'returns each intermediate result'. 

Let's try and reason why it would make sense to return each intermediate result. We know that `Observable`s can be asynchronous. They are wrappers around values that are pushed towards us, either sync or async. Since those values are delivered 'over time' it is very useful in a lot of scenarios to have temporary values.

Let's think about Arrays for a second. Arrays are datastructures. These datastructures are **always sync**. When you get an Array, it already 'holds' all of the values. If we think that the `scan` operator will emit temporary results as values might be delivered over time, it doesn't make sense that there is a `scan` operator for Arrays. Arrays are never async, so temporary calculated values wouldn't have any benefit!

## Conclusion

As Arrays are **sync** datastructures, having an operator/method that emits temporary results when accumulating the values from an Array, doesn't makes sense.
For `Observable`s it does makes sense as the values (are|can be) delivered over time.

















