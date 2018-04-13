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


Polling is a common scenario in a lot of Single Page Applications. You want your user to see the latest data without them taking any actions. In some scenarios, you might even want to display this data realtime. In most cases however, this is overkill and requires changes at the backend of your application. Polling is a really good 'near immediate' alternative.

Polling is something where RxJS really shines. We will look at different polling strategies and how we can implement them.

**Note:** The examples in this post will use Angular but the concepts can be ported everywhere.

- [Simple polling](#simple-polling)
- [Combining polling with refresh button](#polling-and-refresh-button)
- [Polling and reset](#polling-and-reset)
- [Polling when data is resolved](#poll-when-data-is-resolved)

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

<iframe style="width: 100%; height: 500px" src="httsps://stackblitz.com/edit/angular-abcqen?embed=1&file=app/app.component.ts"></iframe>
 
### Polling and refresh button

Sometimes, users can be pretty impatient and want to have the control to fetch the data. We can accomplish this by adding a button that, when clicked, will fetch the data as well. But we want to keep our polling as well.

Lets first try to think reactive on how we can accomplish this. We already have a stream that polls the data. We can create a stream that fetches the data whenever the button is clicked. When we have both of these streams, we can actually just combine them using the `merge` operator to get one stream that is both triggered by the polling and the button click.

We can simply add a button to our example and a click listener. When the button is clicked, we need to convert this click into a stream, since we will need a stream to 'start with'. For this we can leverage a `Subject`.

```typescript
manualRefresh = new Subject();

refreshDataClick() {
    this.manualRefresh.next('');
}
```

Now that we have a stream that is fired every time the button is clicked, we can simply use the same way of working that we had before. Just know, our 'source' stream is not a `timer` but a `subject`.

```typescript
this.manualRefresh
	.pipe(
       concatMap(_ => bitcoin	$),
   );
```

Next thing we need to do is combine both of our streams that can trigger a backend call (and remove the double `concatMap` operator).

```typescript
this.polledBitcoin$ = timer(0, 10000).pipe(
        merge(this.manualRefresh),
        concatMap(_ => bitcoin$),
        map((response: {EUR: {}}) => response.EUR),
        map((EUR: {last: number}) => EUR.last),
      );
```

That's it. Now whenever the button is clicked or the timer triggers, a backend call will be done. 

The live code example can be found here:

<iframe src="https://stackblitz.com/edit/angular-zytccc?embed=1&file=app/app.component.ts" style="height: 500px; width:100%"></iframe>

**Note:** You can open the devtools on the network tab to see the network requests.

### Polling and reset

The previous polling strategy can introduce some unnecessary backend calls. Lets think about the following scenario. Our timer stream triggers every 10s. After the app has started and has been running for 19s, the user clicks the button, triggering a backend call. And after 20s our timer stream fires as well, also triggering a backend call. This means that at both the 19th and the 20th second, we are fetching the data. This might be a little overkill.

Lets think about how we can fix this. We already have a stream that will fetch the data immediately and then again and again with 10s in between. And actually, that's all we need. When we have this stream, and the user clicks the button, we can just restart this stream. Becuase, when we restart this stream, we are fetching the data immediately (which is what we want when the user clicks), and again after 10 seconds. The ASCII marble diagram looks like this:

```
bitcoin:            -(b|)
user clicks:                            C 
trigger$:           0------1------2-----|0------1------2-----|
                    \      \      \      \      \      \      
                     -(b|)  -(b|)  -(b|)  -(b|)  -(b|)  -(b|)
 
 
polledBitcoin$     ----b------b------b------b------b------b-- 
```

In the marble diagram above, 'C' denotes the user click. In that case, we want to unsubscribe from the previous execution of our `trigger$` and execute it again. Lets see how we can do this in the code:

```
manualRefresh$ = new BehaviorSubject('');

this.polledBitcoin$ = this.manualRefresh$.pipe(
      switchMap(_ => timer(0, 10000).pipe(
         concatMap(_ => bitcoin$),
         map((response: {EUR: {}}) => response.EUR),
         map((EUR: {last: number}) => EUR.last),
      )
   )
);
```
First thing we need to change is move from a `Subject` to a `BehaviorSubject`. A `BehaviorSubject` has an inital value and will replay the last value when subscribed to. Here, we are interested in the fact that it has an initial value.

Next thing we do is use this subject to create our `polledBitcoin$`. We wrapped the stream from our previous examples in a `switchMap`. Whenever the `manualRefresh$` emits, this stream will be started. If there was a previous execution still working, this execution will be stopped in favor of a new one. And that's exactly what we need.

Now, whenever the user clicks on the reload button, the data will be fetched and the timer is reset! Nice right!

### Polling when data is resolved

The last polling strategy I want to take a look at is one where we only start a next request after the first one has finished plus 'x' seconds. Lets say we are polling every 5 seconds and at one point, our backend call takes 4 seconds. This would mean that, 1 second after we finally gotten our result, we fetch it again. This might not always be what we want.

To












