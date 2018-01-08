---
layout: post
cover: false
title: Multicasting operators in RxJS
date:   2017-08-20
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS
---

With the arrival of RxJS 5.4 a while back, the RxJS team has given us yet another way to support multicasting in our applications. They introduced the `shareReplay` operator. With this new one around the corner, you might start wondering when to use which multicasting operator. Well, it's your lucky day cause that's what this post is all about.

**TL:DR;** You can find an overview of all the multicasting operators and their properties <a href="#operator-overview">here</a>. 


## What is multicasting

First of all, I would like to go a little deeper into the subject of multicasting. What does this really mean? As you hopefully know, observables can be divided into two categories, hot and cold.
If you subscribe to an observable, you are going to start executing that observable. What this means is the observable will start producing values. When you are working with a cold observable, every new subscription will 'restart' the observables producer. 

**Note**: If you do not know what hot and cold observables mean, you can read this excellent article on the Thoughtram blog <a href="https://blog.thoughtram.io/angular/2016/06/16/cold-vs-hot-observables.html" target="_blank">here</a>.

**Note2:** The fact that an observable is either cold or hot is somewhat debatable as we'll see later on. An observable can also hold properties from both of these states. In the Thoughtram article described above, they point to these observables as being 'warm'.

Let's take a look at an example:

<a class="jsbin-embed" href="http://jsbin.com/jefupu/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

Here we can see an interval observable that will emit 5 values with half a second between them. We subscribe to this observable immediately and again after 1,5 seconds. As you can see, when the second subscription happens, it doesn't get the same values as the first subscription. Instead, it starts with the value '0'. 
We can conclude from this that for every subscription, the observable is 'restarted' and the observable will restart the production of values. 

If we try to put this into a visual representation, it might look a little like this:

<img src="https://www.dropbox.com/s/y9bh74hsiqwy205/Screenshot%202017-09-03%2014.55.00.png?raw=1" width="400">

We can see that the interval observable is 'recreated' when the second subscription occurs.

This might feel a little weird in the beginning, but it gives us the benefit to re-use observables, which is a quite powerful concept once you get the hang of it. It however also introduces some weird side effects. Let's take a look at an example:

<a class="jsbin-embed" href="http://jsbin.com/terapof/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

 
We create an observable, `getLuke$`, which will perform a call to fetch the character of Luke Skywalker from the swapi.co API. We use this as a source to create two new observables. One holds the name of the character, the other one holds the gender of the character. We immediately subscribe to both of the observables. If you open your devtools onto the network tab, you will see that there are actually two network request being performed.

This might seem weird at first, but in fact, it's quite logical. The `getLuke$` observable we created is a cold one. The two new observables we create both use this one as a source. So in fact, subscribing to our `gender$` and `name$` observable, is the equivalent of subscribing to the `getLuke$` observable twice. And, as we have seen above, every subscription to a cold observable, will trigger two executions of the observable, two times the production of values, thus in this case two network requests.

While this behaviour can be useful, sometimes you might want two backend calls, it can also be quite annoying. The problem that we are facing here is that the execution of the observable is restarted on every subscription. While sometimes, we want to share the underlying subscription. Sharing the underlying subscription is what multicasting is all about.

**Note:** If you are multicasting an observable, you have transformed it from a cold to a hot observable.

### Multicasting example

Let's change our example to share the underlying subscription. For this we will use the `share` operator for now. We will investigate all the other ones and their properties later on.

<a class="jsbin-embed" href="http://jsbin.com/higujow/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

If you run this example while opening your devtool's network tab, you can see that there is only one request. That's because the underlying subscription is shared. 
Let's again try to visualize this in a diagram.

<img src="https://www.dropbox.com/s/kcohbr9wa5ax631/Screenshot%202017-09-06%2021.08.08.png?raw=1" width="400">

Here, we can see that the share operator will only subscribe once to the source observable, being the observable that does a backend call, and will multicast the data to all the subscriptions. It acts as a proxy.

## The properties of multicasting

A multicasting operator shares the underlying subscription towards its subscribers. The way they do it can vary quite a lot. Next we are going to go over all of the properties a multicasting operator can have.

### Connectable

One of the ways to share the underlying subscription to multiple subscribers, is by using the `publish` operator. When you call `publish()` on an observable, you get back a `ConnectableObservable`. This is an observable that will subscribe to the source observable once you have called it's `connect()` method. Let's try and put this in a ASCII marble diagram to visualise it better.

```typescript
source observable:         ---a----b----c|
                             -publish()-
connect point:             C
subscriber 1:          ^------a----b-!     
subscriber 2:                   ^--b----c|
```

**Note:** If you don't know the ASCII marble syntax, take a look at the <a href="https://github.com/ReactiveX/rxjs/blob/5.4.2/doc/writing-marble-tests.md#marble-syntax" target="_blank">official documentation</a>

We have a source observable which will emit 3 values, a, b and c. We use the `publish` operator on this cold observable. This will return a `ConnectableObservable`. We have a subscriber that subscribes immediately to this stream, and a subscriber that subscribes after some time.
 
We can see that the first subscription point of subscriber 1, doesn't trigger the source observable to be started. It's only at the time the `ConnectableObservables`'s `connect` method gets called (indicated by the 'C'), that the source observable is started.
When the second subscription happens, the 'a' value has already been passed by the `ConnectableObservable` to all available subscribers at that time, which was only the first subscriber. The second subscriber missed this value. 
When the 'b' value is produced by the source observable, it is passed to both the first and second subscriber. 
Next the second subscriber unsubscribes (denoted by the '!'). So when the source observable emits the last value, c, and completes, only the second subscriber gets these values.

Let's take a look at coding example:

<a class="jsbin-embed" href="http://jsbin.com/wicarev/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

Here we can see an interval observable that will emit 4 values. We use the `publish` operator to create a `publishedInterval` observable. We subscribe to it immediately and we subscribe to it after 2500ms. As you can see, the first subscription will not trigger the interval to be started. It's only when we call it's `connect` method after 600ms that it will start emitting values. 

**Conclusion:** A multicasting operator is connectable when you have to call the `connect` method before it subscribes to the source observable and starts proxying.


### Reference counting

The next property we are going to discuss is reference counting. As we've seen above, when we get a `ConnectableObservable` we need to call `connect()` on it before the source observable is subscribed to. Sometimes, you might want the source observable to be subscribed to as soon as there is at least one subscriber. And that's exactly what you can achieve with reference counting through the `refCount` operator. Let's see what this looks like in a marble diagram.

```typescript
source observable:     --0--1--2--!    --0--1!
                             -publish().refCount()-
subscriber 1:          ^-0--1!
subscriber 2:             ^-1--2--!
subscriber 3:                          ^-0--1!
```

We have a source observable that will emit values with some time in between. As soon as the first subscription happens, the source observable is started. When the second subscription happens, the source observable is still emitting values and it will get the same values as the first subscription. When the first subscription stops, the source observable is not unsubscribed to, but when the second one stops, it is. 
The `refCount` operator will count the number of subscriptions. As soon as this number is 1, it will subscribe to the source observable and as long as this number stays 1 or higher, the source observable is subscribed to. If this number drops to 0, it unsubscribes from the source observable. 
When the number rises back from 0 to 1, as it is with our third subscriber, the source observable is resubscribed to.

Let's take a look at some code:

<a class="jsbin-embed" href="http://jsbin.com/vujozac/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

We have an interval observable that `publish().refCount()` is applied to. This newly created observable is subscribed to. When this happens, the source observable, our interval observable, is immediately subscribed to. 
After 700ms, we subscribe a second time. At this moment, both subscriptions get the same values.
When the first subscriber unsubscribes, after 1100ms. It doesn't influence the source observable. This is because the reference counting is still on 1. When the second subscriber stops listening, after 1700ms, the source observable is unsubscribed to because the reference counting dropped to 0. 
When we resubscribe a third time, the source is subscribed to again.

**Note:** Using `publish().refCount()` is a quite common operation. For that reason, the `share()` alias was introduced which is the exact same thing.

**Conclusion:** An observable is reference counting when it subscribes as soon as there is a single subscriber and stops when there are no more subscribers. 

### Replayable

If you subscribe to a hot observable, you might have missed some values. In some cases, this might not be what you want. You might want to get at least the latest emitted value before you subscribed or the latest x values that were emitted before you subscribed. Luckily, there is a way to do that. 

Let's first create an ASCII marble diagram to visualise what we want:

```typescript
source observable:     ---a----b-------c----d----e|
                             -shareReplay(2)-
subscriber 1:          ^--a----b!     
subscriber 2:                    ^(ab)-c----d----e|
```
 
In this scenario, we are using the `shareReplay` operator. We are subscribing to the created observable twice. When the second subscription happens, the source stream has already emitted two values. When the second subscription happens, it normally would have missed these two values. But because we use the `shareReplay` operator we get these two values. We passed '2' to the operator which means that it will replay the last two values before the subscription.

Let's take a look at an example:

<a class="jsbin-embed" href="http://jsbin.com/qumego/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

We have an interval observable, to which the `shareReplay` operator is applied with parameter '2'. We subscribe to it immediately and once after some values have already been passed. We can see that, as soon as the second subscription happens, it also receives the last two values that were emitted before the subscription.

**Conclusion:** A multicasting operator is replayable when it emits the 'x' latest values to a new subscriber.

### Repeatable

In the previous examples, we were dealing with a source observable that completed. This could mean that if we subscribe to an observable that is multicasted or hot, for which the source has completed it will never get any values. For this reason, there are also hot observables that are repeatable. Let's look at a ASCII marble diagram that represents this:

```typescript
source observable:     ---a----b|    ---a----b|    
                             -share()-
subscriber 1:          ^--a----b|     
subscriber 2:                   	   ^---a----b|
```

We have a source observable that we are applying the `share` operator to. When we have a first subscriber, the source observable is started and the first subscriber get's all the values. But by the time the second one subscribes, the source observable has already completed. In that case, at least for the `share`, the source observable is resubscribed to and the second observable will get the same values (remember the source observable in this case is a cold one so for every new subscription, the observable is restarted).

Let's take a look at an example:

<a class="jsbin-embed" href="http://jsbin.com/doyojenatu/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

Here we have an interval observable where we apply the `share` operator to. It will emit 3 values in 1.5 seconds and then complete. We subscribe to it once immediately and again after 3 seconds. If we look at the output, we can see that both the subscriptions get the same values. From this we can conclude that the source observable was repeated.

**Conclusion:** A multicasting operator is repeatable when it resubscribes to the source observable when there is a new subscription and the source observable had already completed. It re-executes the source observable.

### Retryable

As stated before, a multicasting operator will share the underlying subscription towards it's subscribers and acts as a proxy. But what happens when this source observable throws an error? There are multicasting operators that will retry subscribing to the source observable when it threw an error. Let's put this into a marble diagram:


```typescript
source observable:     ---#            ---#
                             -shareReplay()-
subscriber 1:          ^--#
subscriber 2:                   	   ^--#
```

Here we have a source observable onto which the `shareReplay` operator is applied. When the first subscriber starts listening to it, the source observable will be subscribed to. Here, it will throw an error after some time which is send to the first subscriber.
A little while later the observable is resubscribed to by a second subscriber. This will start a new invocation of the source observable. This one will have the same effect as the first subscription. In a real life scenario, the first invocation might fail, but this doesn't necessarily mean that the second will. In those scenario's, retrying can be very useful.

Let's look at an example:

<a class="jsbin-embed" href="http://jsbin.com/diroqe/embed?js,console">JS Bin on jsbin.com</a><script src="http://static.jsbin.com/js/embed.min.js?4.0.4"></script>

We have an observable `throw$` that will, once subscribed to, will throw an error. We use the `do` operator to see if it was invoked. We apply the `shareReplay` operator to this `throw$` observable. 
We subscribe immediately and after three seconds. We can see that, even though the first subscriber gets an error, as soon as the second one subscribes, the source observable is resubscribed to by the `shareReplay` operator. This makes it retryable.

**Conclusion:** A multicasting operator is retryable when it resubscribes to the source observable when there is a new subscription and the source observable has errored before.

## Operator overview

Time to give an overview of all the multicasting operators and their respective properties:
{::comment}
|                     | Multicasting 	                | Connectable                     | Reference counting*             | Replayable                      | Repeatable                      | Retryable                       |
| ------------------- | :-----------------------------: | :------------------------------:| :-----------------------------: | :-----------------------------: |:------------------------------: |:------------------------------: |
| share()             | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox" checked> | <input type="checkbox" checked> |
| shareReplay()       | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox" checked> | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox" checked> |
| publish()           | <input type="checkbox" checked> | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         |
| publishReplay()     | <input type="checkbox" checked> | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox">         |
| publishBehaviour()  | <input type="checkbox" checked> | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         |
| publishLast()       | <input type="checkbox" checked> | <input type="checkbox" checked> | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         | <input type="checkbox">         |
{:/comment}
![tableOverview](https://www.dropbox.com/s/6kufxt28eca9e94/Screenshot%202018-01-08%2019.26.33.png?raw=1)

<nowiki>*Every multicasting operator which is connectable can also have the `refCount` operator applied to. In this table however, every operator that has 'Reference counting' checked, has reference counting implicitgit ly applied in the implementation of the operator.</nowiki>

