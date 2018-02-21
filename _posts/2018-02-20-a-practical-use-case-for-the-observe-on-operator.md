---
title: A practical use case for the observeOn operator
published: true
author: kwintenp
description: This article shows a practical use case for the observe on operator
layout: post
navigation: True
date:   2018-02-20
tags: Angular
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
disqus: true
cover: 'assets/images/cover/cover4.jpg'
---
A few weeks ago, I needed to implement a use case where the user could enable a certain feature when he triple clicked on the screen. Leveraging the power of RxJS, I thought it would only take a couple of minutes to create. Turns out it was a little more difficult than expected. I even had to use the 'observeOn' operator to get everything working. I discovered a pattern where the operator would become usefull in other scenarios as well. Let me show you how I implemented the triple click and what the observeOn operator was used for.

// List all the operators they will learn

// Add a TL:DR;

### Creating a click stream

Creating a stream of clicks is fairly easy. We can use the 'fromEvent' operator for that. Let's look at the code:

```typescript
const clicks$ = Rx.Observable.fromEvent(document, 'click')
  .mapTo('click');
```

The 'fromEvent' operator takes the element you want to add an event listener to and the event name. Using the 'mapTo' operator, we transform the event into a string, which is a little more convenient when we log out the event.

### Listen for clicks close to each other

Multi clicks only happen when the clicks happen close together. We could for example say that we want to 'catch' all the clicks that happen until the user stops clicking for 250ms. To accomplish this, we can use two operators, 'buffer' and 'debounceTime'. 

'Buffer' allows us to catch all events until a certain event occurs. This way, we can 'capture' all clicks until the user stops clicking. This will return an array with the 'click' events which we can use to filter out the triple clicks.

```typescript
clicks$.buffer(//streamThatFiresWhenUserStopsClicking)
	.filter(x => x.length > 2);
```

'DebounceTime' allows us to know when the user stopped clicking. 

```typescript
clicks$.debounceTime(250);
```
Combined this looks like this:

```typescript
const clicks$ = Rx.Observable.fromEvent(document, 'click')
  .mapTo('click');

clicks$.buffer(clicks$.debounceTime(250))
  .filter(x => x.length > 2);
  .subscribe(val => console.log('TRIPLE CLICK DETECTED'));
```

### Emit when the user has clicked exactly three times

Using the 'debounceTime' operator, we kind of introduced a problem. If the user keeps on clicking, our 'trigger' in the buffer will not fire. The trigger shouldn't only fire when the user stopped clicking for 250ms, but also when the user has already clicked three times.

Let's first create a stream that will trigger when the user has clicked three times. To do that, we can use the 'scan' operator. 'Scan' allows us to count the number of clicks by counting the number of events passed and filtering when the count is lower than 3. Code wise it looks like this:

```typescript
clicks$.scan((acc, curr) => ++acc, 0)
		.filter(x => x > 2)
```

Ok, now that we have a stream that fires if the user stops clicking and one that fires if there are three clicks, we can compose a new trigger. Since it's a one or other situation, we can use the 'merge' operator to combine the streams.


```typescript
const trigger$ = clicks$.scan((acc, curr) => ++acc, 0)
       .filter(x => x > 2)
       .merge(clicks$.debounceTime(250));
```

This makes the entire code looks like this atm:

```typescript 
const clicks$ = Rx.Observable.fromEvent(document, 'click')
  .mapTo('click');

const trigger$ = clicks$.scan((acc, curr) => ++acc, 0)
       .filter(x => x > 2)
       .merge(clicks$.debounceTime(250));

clicks$.buffer(trigger$)
  .filter(x => x.length > 2)
  .subscribe(val => console.log('TRIPLE CLICK DETECTED'));
```

You can try it out here:
<a class="jsbin-embed" href="http://jsbin.com/cakigif/embed?js,console,output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.1.2"></script>

Hmmm, it seems to work the first time, but not after that. The reason is quite logical actually. We are not resetting the counter in our 'scan' operator. After the very first 3 clicks, that inner counter is constantly rising. Since the next part in our trigger is the 'filter' which checks if the count is bigger than 3, which it is, our trigger is fired every time. We need to reset our `trigger$`.

### Resetting the trigger stream
Resetting or restarting a stream is something we can do with the 'switchMap' operator. A 'switchMap' operator will map a value to a stream and start listening to that stream. As soon as the operator receives a new event, it will stop listening to the previous stream and subscribe to a new one.

Thinking about our example, we want to reset our stream every time the 'buffer' has fired because this is the moment the user either stopped clicking or clicked 3 times. We need to create a stream that fires on those events. Since this creates some circular dependencies we will have to use a subject for that. Let's look at the code:

```typescript
const reset$ = new Rx.BehaviorSubject('');

const trigger$ = reset$.switchMap(
     () => clicks$.scan((acc, curr) => ++acc, 0)
                  .filter(x => x > 2)
                  .merge(clicks$.debounceTime(250))
);

clicks$.buffer(trigger$)
  .do(() => reset$.next(''))
  .filter(x => x.length > 2)
  .subscribe(val => console.log('TRIPLE CLICK DETECTED'));
```

We use a 'BehaviorSubject' because this has an initial value. This will make sure that our `trigger$` gets started immediately. After our 'buffer', we 'next' the `reset$` subject so our previous trigger stream gets unsubscribed and a new one is started for the next execution.

You can try it out here: 
<a class="jsbin-embed" href="http://jsbin.com/yehoyen/embed?js,console,output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.1.2"></script>

If you try it out, you'll probably notice that it doesn't work the first time. After the first time, it does work as expected. The reason here is because the events are being handled in the wrong order.

### Using observeOn to order event listeners

Let's investigate what is happening here by writing in out in different steps. For this, I added some logs to the example code as you can see here: 

<a class="jsbin-embed" href="http://jsbin.com/femezop/embed?js,console,output">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.1.2"></script>

1. The stream is created.
2. The stream is subscribed.
3. We are subscribing to the `clicks$` twice. Once via our `trigger$` and once in our 'main' stream.
4. As the `click$` is cold, both subscriptions register an event listener.
5. When you click once, you can see that the 'buffer click' is triggered before the 'main click'. You can see this from the log statement. 
6. When you've click three times, the trigger stream will fire since you have clicked three times.
7. This will cause the 'buffer' to fire and emit an event. BUT, it will emit an array with only '2' clicks. The click listener in the 'main stream' has not fired yet, the 'buffer click' was first!
8. Next we filter out all the arrays that have a length smaller than 3, hence our first 'triple click' is not detected.
9. As our `trigger$` gets reset, the event listener is removed and added again. The side effect from this action is that the 'main click' is now the first.
10. Since for subsequent 'triple clicks' the 'main click' is triggered first, the buffer will emit array events with the correct number of clicks and the stream works.

To fix it, we would need to be 100% sure that the event listener for our 'main' stream always fires BEFORE the click in our `trigger$`. Here comes the 'observeOn' operator into play. By adding the operator we can make sure a certain event only gets handled on a certain scheduler. Let's see how we can change the code:












