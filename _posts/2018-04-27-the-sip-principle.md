---
layout: post
title: Thinking reactive with the SIP principle
date:   2018-06-30
subclass: 'post'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: brechtbilliet
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---
A few months back we released [RxJS best practices in Angular](https://blog.strongbrew.io/rxjs-best-practices-in-angular/) and a while before that [Thinking reactively in Angular and RxJS](https://blog.strongbrew.io/thinking-reactively-in-angular-and-rxjs/). 
Both of these articles are focussing on "trying to make the mind switch towards reactive programming".

However, sometimes we like to have structured opinionated ways of tackling problems, especially when things become complex. We like a roadmap of some kind, something to fall back on, something to guide us through these complex reactive scenarios.

While writing RxJS code for small pragmatic solutions can be super easy, it might become complex when combining multiple streams or doing other advanced stuff. 

We as StrongBrew are huge fans of reactive programming and we use our reactive mindset in Angular on a daily basis.
In this article we will learn a principle that helps us to tackle very complex RxJS situations in an opinionated structured way.

The code of this article is written with [Angular](https://angular.io) but the concepts can be used with every framework.

## The situation

We are going to build an application to search for starships in the [swapi api](https://swapi.co). The application counts a few features:
- It has to load data on page load
- The user can search for starships by entering text in the searchbox
- The user can load starships by a chosen model
- The user can load starships by a random model
- There is a loading indicator that needs to be shown when the data is being loaded
- Previous XHR calls should be canceled to avoid race conditions
- We want to filter the results by the number of passengers allowed on the ship. **Note: This is a clientside filter**

As we can see, there is quite a lot of asynchronous logic going on here, and if we would implement this in an imperative way, it would be pretty hard to keep it simple. However, the application can easily be written with the use of RxJS. In this application we don't want to think in actions anymore, we want to think in streams of data. **Everything is a stream!**

## The SIP principle
With StrongBrew, we came up with a simple principle of handling this reactive mindset. We call it the **SIP principle**.
The SIP principle stands for the following:
- S: Source streams
- I: Intermediate streams
- P: Presentation streams

### Source streams

These streams will contain all the user interaction. They are generally bound to the outputs of our dumb components. A source stream could also contain real-time data, but we won't cover that part in this article.
Let's have a look at our application and look for the source streams there:
(the left column pretty much contains all the user interaction)
![Source streams](/assets/images/posts/sip-principle/source-streams.png)

After executing the first step of the SIP principle we have found 4 source streams.
`searchTerm$`, `selectedModel$`, `randomModel$` and `numberOfPassengers$`.

What's important to note here is that we named these streams as **streams of data** rather than naming them as streams of actions. We want to stop thinking in actions and start thinking in streams of data, remember? That's why we didn't name the first stream `search$` (which would be an action), we rather named it `searchTerm$` so we know it contains a search term.

Source streams are **mostly subjects** or streams that come from the framework, e.g. a routing params stream.

In our application the source streams will look like this:

```typescript
searchTerm$ = new ReplaySubject<string>(1);
selectedModel$ = new ReplaySubject<string>(1);
randomModel$ = new ReplaySubject<string>(1);
// needs an initial value
numberOfPassengers$ = new BehaviorSubject(1000000); 
```

These subjects will be populated by the outputs as shown below:

```html
<sidebar 
    (search)="searchTerm$.next($event)"
    (selectModel)="selectedModel$.next($event)"
    (randomModel)="randomModel$.next($event)"
    (changeNumberOfPassengers)="numberOfPassengers$.next($event)"
>
</sidebar>
...
````

### Presentation streams

After finding the source streams we need to find the presentation streams.
These are the streams that our template needs to render properly. These are quite easy to find. We just have to look at the template and see which inputs our components expect. We can have a look at our template. (The outputs are stripped for readability purposes).

```html
  <sidebar class="sidebar" 
    [models]="fixedModels" 
    [numberOfPassengers]=""
  >
  </sidebar>
  <div class="main">
    <starship-list 
        [starships]=""
        [loading]="">
    </starship-list>
  </div>
```

We can instantly see that we need 3 presentation streams:
We need the number of passengers, the starships that need to be shown and whether the application is loading or not. Let's fill in the blanks, shall we?!

```html
  <sidebar class="sidebar" 
    [models]="fixedModels" 
    [numberOfPassengers]="numberOfPassengers$|async"
  >
  </sidebar>
  <div class="main">
    <starship-list 
        [starships]="filteredResults$|async"
        [loading]="loading$|async">
    </starship-list>
  </div>
```

So after step 2 we have found the following presentation streams: `numberOfPassengers$`, `filteredResults$` and `loading$`.

### Starting with the SIP diagram

Let's visualize these streams by creating a SIP diagram:

![Source streams, presentation streams](/assets/images/posts/sip-principle/sp.png)

The goal is to calculate the presentation streams, based on the source streams. There are 2 presentation streams that we need to calculate: `filteredResults$` and `loading$`. We don't need to calculate `numberOfPassengers$` since it's the same stream as the source stream.

Let's start with the `filteredResults$`. The `filteredResults$` is dependent on `searchTerm$`, `selectedModel$`, `randomModel$` and `numberOfPassengers$`. To make this calculation easier we can use intermediate streams.

### Intermediate streams

Intermediate streams are streams that are used to make the bridge between the source streams and presentation streams easier.
Let's create a `query$` stream and a `results$` stream to make the calculation easier.


![SIP 1](/assets/images/posts/sip-principle/sip1.png)

We will use custom marble diagrams to visualize the different parts of the SIP diagram throughout this article.

#### Calculating the query$

The first intermediate stream that we have to create is the `query$` which is simply a merge from the `searchTerm$`, `selectedModel$` and `randomModel$`.


![Query](/assets/images/posts/sip-principle/query.png)

As we can see these 3 streams are being merged into one new `query$`. We don't have to worry about when a user searches or selects a model, or even requests a random model. We only care about that simple stream of data, the `query$`.

#### Calculating the results$

Every time the `query$` gets a new value we want to fetch data from the API. For that we will use the `switchMap` operator.


![Results](/assets/images/posts/sip-principle/results.png)

#### Calculating filteredResults$

We are ready to finish up the `filteredResults$` stream.
If we look back at the previous SIP diagram we can see that we can create that stream by combining the `results$` and the `numberOfPassengers$`.

![Filtered results](/assets/images/posts/sip-principle/filteredResults.png)

#### The loading$

The next presentation stream that we want to create is called the `loading$`. Let's update the SIP diagram accordingly. The `loading$` is based on the `query$` and the `results$`.

![sip 2](/assets/images/posts/sip-principle/sip2.png)

Every time the `query$` gets a new value the `loading$` should get the value `true`. Everytime the `result$` gets a new value the `loading$` should get the value `false`.
So if we map every value of the `query$` to `true` and if we map every value of the `results$` to `false`, and merge those 2, we have created our `loading$` stream.

Let's create one last marble diagram for that.
![Loading](/assets/images/posts/sip-principle/loading.png)

### What do we need to share?

We have came a long way, we have created the complete SIP diagram. We have visualized all the streams by creating marble diagrams, but if we look closely at the SIP diagram we might notice a problem.
There are 2 arrows leaving the `query$` and 2 arrows leaving the `results$`. Every arrow stands for a subscription on the observable where the arrow starts.
When working with cold observables, the producer function for that observable is executed every time we subscribe. Since `query$` is a hot stream, it doesn't really matter, but `results$` will trigger an HTTP call every time it gets subscribed to. In short: the subscription on `loading$` and `filteredResults$` will trigger a subscription on `results$` twice. The SIP diagram show us which subscriptions need to be shared.

![sip3](/assets/images/posts/sip-principle/sip3.png)

## Check it out

The SIP diagram is complete now and we can start coding. Since this article is really about the SIP principle I won't explain the code in detail. However, you can find the complete code in the StackBlitz below.
<iframe src="https://stackblitz.com/edit/sip-principle?embed=1&file=app/app.component.ts" style="width: 100%; height: 500px"></iframe>

## Conclusion

Before starting with implementing complex RxJS screens. Take a whiteboard and draw the flow first. The SIP principle that we created works for us but isn't the only way to go of course.

If you check the code inside the StackBlitz, you might be surprised about the amount of lines of code that we need to create this application. We have only a few lines of real logic, and we have covered most corner cases by thinking reactive.

Because of the use of the `async` pipe we don't need to unsubscribe from any stream manually since the `async` pipe does that for us.

## Special thanks

[Jurgen van de Moere](https://twitter.com/jvandemo) for helping us with finding the right acronym.

And the awesome reviewers:
- [Manfred Steyer](https://twitter.com/manfredsteyer)
- [Jan-Niklas Wortmann](https://twitter.com/niklas_wortmann)
- [Tim Deschryver](https://twitter.com/tim_deschryver)
- [David Müllerchen](https://twitter.com/webdave_de)