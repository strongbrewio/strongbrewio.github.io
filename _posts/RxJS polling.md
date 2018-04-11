---
title: RxJS polling
published: true
author: kwintenp
description: An overview of different polling strategies with RxJS
layout: post
navigation: True
date:   2018-04-11
tags: RxJS Angular
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover10.jpg'
---


### RxJS polling

Polling is a common scenario in a lot of Single Page Applications. You want your user to see the latest data without them taking any actions. In some scenarios, you might even want to display this data realtime. In most cases however, this is overkill and requires changes at the backend of your application. Polling is a really good 'near immediate' alternative.

Polling is something where RxJS really shines. We will look at different polling strategies and how we can implement them.

**Note:** The examples in this post will use Angular but the concepts can be ported everywhere.

### Simple polling

First we will take a look at a simple example where we want to fetch new data every 5 seconds. Lets first try and think about what we need. 

- a backend call
- a trigger that tells us when we need to execute our backend call

#### Backend call

Executing a backend call is easy. We can create a stream that will execute a backend call when subscribed to like this.

```typescript
const bitcoin$ = this.http.get('https://blockchain.info/ticker');
```

#### Trigger

Next thing we need is a trigger that will tell us when it is time to fetch our data. In a world without RxJS we would probably use `setInterval`. This method allows us to pass it a callback that gets executed every 'x' seconds. 
With RxJS however, we have to change the way we think. We can no longer think in terms of callbacks, we have to think in terms of streams. If we apply this to the trigger we need, we want a stream that fires every 'x' seconds. 
Drawn in a ASCII marble diagram, this is what we want:

```
----1----2----3----4----5...
```

RxJS has a static `interval` method that will create this streams for us. We can pass it a number which will denote the time between the events.

```typescript
const trigger$ = interval(1000);
```

This is not enough however. Our trigger stream should also trigger at start time. Otherwise, we would only fetch data after '1000ms' (with the example above).

RxJS provides another static method, 'timer', that will help us to create the following stream:

```
0----1----2----3----4----5...
```

Code wise, this looks like this:

```typescript
const trigger$ = timer(0, 1000);
```

#### Combine to polling stream

Now we have the two streams that we need, it is time to combine them. If you think about it, we basically want to re-execute our `bitcoin$` to refetch the data, every time our `trigger$` fires. We want to map our trigger value to another observable/async action. To do that, we need to use a flattening operator. As flattening operators are not part of this post, you can read more about them <a href="https://blog.angularindepth.com/switchmap-bugs-b6de69155524" target="_blank">here</a>. 

In our case, we are going to use the `concatMap` operator. This operator will execute all the `bitcoin$` without cancelling them. Let's take a look at the code:

```typescript
this.polledBitcoin$ = timer(0, 1000).pipe(
        concatMap(_ => bitcoin$),
        map((response: {EUR: {}}) => response.EUR),
        map((EUR: {last: number}) => EUR.last),
      );
```
We create a new stream, `this.polledBitcoin$` by mapping every event that the `trigger$` emits to our `bitcoin$`. The `concatMap` operator will subscribe to the the `bitcoin$` internally and emit the result of that stream as events on the `polledBitcoin$`.

When we draw this out into a ASCII marble diagram, it looks like this:

```
bitcoin:            -(b|)
trigger$:           0------1------2------3------4------5...
                    \      \      \      \      \      \
                     -(b|)  -(b|)  -(b|)  -(b|)  -(b|)  -(b|)
 
 
polledBitcoin$     ----b------b------b------b------b------b 
```

We have our `bitcoin$` that will, when subscribed to, take some time, and then emit the event and complete.

We have our `trigger$` where we map the values to the `bitcoin$`. The `concatMap` operator flattens the result and we get our `polledBitcoin$`. A stream that will fetch the value of bitcoin every second.

The live example can be found here:

<iframe src="https://stackblitz.com/edit/angular-abcqen?embed=1&file=app/app.component.ts"></iframe>









