---
layout: post
title: Why there is no array scan operator
published: true
description: Why isn't the scan operator existing with arrays
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover3.jpg'
---

A while ago, during a training I was explaining the `scan` operator in RxJS and how you can use it to accumulate a 'calculated' value over time. At some point one of the participants asked why there isn't a `scan` operator for `Arrays`? They both have a `reduce` method, right?

And actually it's a really interesting question which showed he didn't understand important differences between `Arrays` and `Observables`.

## What does `scan` do in RxJS

Let's take a look at the definition of the scan operator (from the ReactiveX.io) site:

> Applies an accumulator function over the source Observable, and returns each intermediate result, with an optional seed value.

If you don't know what the `scan` operator does, this definition might not make that much sense. Let's first look at an example on how we can use the operator and then revisit this.

Let's say that we have an `Observable<number>` and we want to visualise the sum of all of the numbers on the screen. This is something we can use the `scan` operator for.  

```
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

On the first run, the initial value will be used (second parameter to the `scan` operator) and the first value from the `Observable`. The calculated value will be 1 (as per our function implementation). 

acc: 0, cur: 1 -> 1

Next time the function gets called, the `acc` is going to be the previously returned value and the `cur` is the next value from the `Observable`. This process repeats for every value emitted by the `Observable`.

acc: 1, 	cur: 2 -> 3
acc: 3, 	cur: 3 -> 6
acc: 6, 	cur: 4 -> 10
acc: 10, 	cur: 5 -> 15
acc: 15, 	cur: 6 -> 21



https://stackblitz.com/edit/rxjs-ylayki




















