---		
layout: post		
title: A scalable angular architecture (part 2)
published: true
description: A while ago I released a blogpost called "A scalable angular 2 architecture" which was all about creating large scale enterprise webapplications. In this article we will cover what sandboxes are really about and why I introduced them in the first place.
author: brechtbilliet
comments: true
date: 2017-08-13
subclass: 'post'
categories: 'casper'
disqus: true
tags: Angular
cover: 'assets/images/cover/cover8.jpg'
---

# The sandbox pattern

## Introduction
A while ago I released a blogpost called "[A scalable angular 2 architecture](http://blog.brecht.io/A-scalable-angular2-architecture/)" which was all about creating large scale enterprise webapplications. The title should actually have been **"A scalable angular architecture"** (It's just angular) or even, **"A scalable SPA architecture"** since we can use the same architecture for [React](https://facebook.github.io/react/) or [Vue.js](https://vuejs.org).

It's been over a year now, since I've been introducing that specific architecture in companies and projects. Nowadays, I know quite a few people that are using this particular architecture in their projects.
Often, I ask developers about how they feel about this SPA-architecture and the feedback is mostly this:
They believe its a **clear and opiniated way** to structure their code, and they have no trouble, finding their way into the codebase.

A big part of the [article](http://blog.brecht.io/A-scalable-angular2-architecture/) in question, was about explaining the so called "sandboxes", a personal terminology for a very specific kind of facade.
Since writing that [article](http://blog.brecht.io/A-scalable-angular2-architecture/) raised a bunch of questions about these "sandboxes", it seemed like a good idea to write a followup article.

In this article we will cover what sandboxes are really about and why I introduced them in the first place.

**Note: This architecture is only one way of structuring your applications and the statements being made in this article are completely personal and might be opiniated.**

## What is a sandbox in a nutshell

In real life it's a place where we can put our children together with a limited set of toys. That way, they can only play with those toys and have fun, and we as parents wouldn't have to worry about them running around in the garden.

In this architecture a sandbox is a controlled environment, and the only place where components can consume pieces of the application.
It's the only interface for our smart components to communicate with the rest of the application. It's a facade that abstracts away logic from our components. It does have very specific logic though.

## Here is why I introduced sandboxes

One of the most challenging parts of creating big clientside javascript applications is encapsulation and dividing responsibilities. Having a decent encapsulated structure where every component, service, class has a clear reason of existence, makes it easy for us as developers to maintain the code and develop new features. 

When writing single-page-applications one could face the following challenges:

### Challenge 1: How do we structure components?

Since this topic is covered in the [previous article](http://blog.brecht.io/A-scalable-angular2-architecture/), we won't do a deepdive in this subject.
We use the smart/dumb component approach to structure our components. The dumb components mostly have presentational purposes and the smart components interact with the rest of the application.

### Challenge 2: How do we handle statemanagement?

Another topic covered by the [previous article](http://blog.brecht.io/A-scalable-angular2-architecture/) is how handle our statemanagement. In short: we use @ngrx/store, or another redux implementation that embraces reactive programming. 

### Challenge 3

The first two challenges are met: we have a clear and structured presentational layer and because of the fact that we use @ngrx/store (or any statemanagement tool for that matter), we have a clear way of managing state. We can optimise performance with immutable data and still have clear unidirectional dataflow.

However, there are still a few pieces of the puzzle missing, which might raise the following questions:

#### How do we separate the presentation-layers from the rest of the application?

If we inject whatever we want in our smart components, their constructors might get huge in no-time. Do those smart components really need to know everything about the application? Do they need to know where everything lives and what service it should call from which module in the application? Wouldn't it be easier if they had an interface they could talk to, that just handles things for them?
It would certainly help to decouple angular modules from each other, and keep the responsibility of those smart components clear and compact. **This is something a sandbox might be able to help us with.**

### Do our components need to know about redux?

Redux is an awesome library/principle that helps us ace statemanagement, but it's a really heavy dependency if you don't manage it carefully. What if your **very specific** redux actions would be everywhere in your components and services? In that case your codebase would be completely affected by it and it would be very hard to use another statemanagement tool in the future. It would be impossible to share that code in non-redux applications.
It would be a huge depencency... What if you like to switch to firebase for instance, or MOBX, or even write your own statemanagement tool. You would need to refactor your whole application. The sole principle of redux is to manage state, and I prefer to use it that way. That is also why I don't really use effects, thunk or saga. (That's a personal preference)
For me this means: **Use redux only to manage state, not to trigger backend actions etc**
One could even argue that the action-dispatching and state-selecting should be combined in central places in your appliction.
 **Let's add that logic to the sandboxes as well.**

#### Do our HTTP services need to know about a redux implementation?

Well, as the name already reveals. An HTTP service is all about doing HTTP communication and returning asynchronous objects to the components. An HTTP service should have no notion about redux at all nor any other statemanagement tool. Its sole purpose is fetching data and returning it. Let's take a look at the following example for instance. We fetch an array of wines from the backend and we want to persist that in a redux store. This is an example of an HTTP service that is being abused to achieve that:

```typescript
class WineService {
	// bad
	fetchWines(): void {
		this.api.get('url')
			.subscribe(wines => 
				this.store.dispatch({type: 'SET_WINES', payload: {wines}});
	}
}
```

This code doesn't belong here, it's not the responsibility of that service. The question is where does it belong?
Does it belong in the component? They would get dirty and reduxy in no-time...
**Again, this would belong to the sandbox!*.

An http service should look like this:

```typescript
class WineService {
	// good
	fetchWines(): Observable<any> {
		return this.api.get('url')
	}
}
```

#### On what place do we dispatch actions to the store?

Err..., the sandbox =)

#### Where do we handle optimistic updates

Again, that would be the sandbox

#### How to NOT make your whole application smell like REDUX

Components and services should not know about redux, so the store interaction should be in a very specific place as well.

I bet you saw it coming, but YES! That's what a sandbox is all about! It's about separation of concerns and having a very specific interface to handle this very specific logic.

## What does a sandbox do?

A sandbox has a very specific responsibility:

- It lets your module communicate with other modules without that module having to know about the rest of the application
- It sends Redux actions
- It exposes streams of data (coming from Redux, Firebase or other technologies)
- It handles optimistic updates
- It keeps the containers stupid enough

## What's very specific on a sandbox?

- They handle what the container tells them to handle, without the container having to know how...
- They interact between modules
- They handle a lot of redux (or similar technology) logic
- The functions mostly have a void return-type, unless we want to be able to cancel HTTP calls
- They expose observables/streams
- It mostly contains redundant code, but no redundant logic (it's all about KISS instead of DRY)
- Mostly every module has a sandbox, unless it contains a huge amount of logic (in that case we could implement a sandbox for every container)
- It does not contain business logic, ever...
- It mostly doesn't contain if-statements, that would be business logic right?!
- It gives us a clear overview of the responsabilities of a module.

## What doesn't a sandbox do?

- It doesn't contain business logic. It contains sandbox logic (see above).
- It doesn't contain presentation logic, like routing etc.
- It doesn't do HTTP calls directly, it delegates to http services.
- It doesn't let your components do whatever they want =)

## An example

```typescript
@Injectable()
export class StockSandbox {
	// these are the store select statements
	// It's pretty dirty to have them all over our components
	// Our components just want to get streams of data, no matter where they come from
	wines$ = this.store.select(state => state.wines);
	isAuthenticated$ = this.store.select(state => state.authentication.isAuthenticated);

	constructor(
		private store: Store<ApplicationState>, 
		private stockService: StockService,
		private fooService: FooService) {
	}

	addWine(wine: Wine): void {
		// use the stockservice to add a wine
		// and when it's done handle our statemanagement
		this.stockService.add(wine).subscribe((wine: Wine) => {
			this.store.dispatch(new AddWine(wine));
		}, () => this.handleError());
	}

	 removeWine(wine: Wine): void {
		// removing a wine can be done optimistically!
		// this would certainly improve the performance and snappyness of our app
		// this would be sandbox logic
		let action = new RemoveWine(wine._id);
		this.store.dispatch(action);
		this.stockService.remove(wine).subscribe(
			() => {}, 
			// if the call failed, we have to undo an action. This is sandbox logic as well!
			() => this.store.dispatch({type: UNDO_ACTION, payload: action}););
	}

	fetchWine(id: string): Observable<Wine> {
		// simple delegation of fetching something: Sandbox logic
		return this.stockService.fetchWine(id).share();
	}

	notifyAnotherModule(): void {
		// our components should NOT know where fooService is, or what it does
		// it should just tell its sandbox to handle a specific action
		this.fooService.doSomething();
	}
}
```

Look how clean this smart component has become:
```typescript
export class StockPageContainer {
    wines$ = this.sb.wines$; // Does this stream comes from Redux? or Firebase (I don't need to know)
    numberOfWines$ = this.wines$.map(wines => sumBy(wines, (wine: Wine) => wine.inStock));

	// a clean constructor makes it easy to test.
	// Util dependencies or presentational dependencies like a router
	// do not belong in the sandbox.
	// They belong right here in the component, in the presentational layer.
    constructor(private sb: StockSandbox, private router: Router) {
    }

    onRemove(wine: Wine): void {
		// Hey Sandbox! Remove the wine please, I don't care how you do it
		// I don't even care that you do optimistic updates, it's not my business
        this.sb.removeWine(wine);
    }

    notifyAnotherModule(): void {
		// I don't know what will happen, but that's okay... It's not my responsability
        this.sb.notifyAnotherModule(); 
    }
}

```


I hope this clarifies the purpose of the sandbox and why I have introduced them in my projects.
If you still have questions about this approach, don't hesitate to contact me =)

### Special thanks
Special thanks to [Manfred Steyer (@manfredsteyer)]('https://twitter.com/ManfredSteyer') and [Juri Strumpflohner (@juristr)](https://twitter.com/juristr) for reviewing this article!



