---
layout: post
title: RxJS patterns - Conditionally executing work
published: true
description: RxJS patterns conditionally executing work
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---

This post is part of a series of blogpost on different RxJS patterns that I use quite often. Here are the other ones:

- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">Restarting work</a>
- <a href="https://blog.strongbrew.io/xjs-patterns-mapping-function-calls-to-streams/" target="_blank">mapping a function to a stream</a>
- <a href="https://blog.strongbrew.io/rxjs-patterns-prematurely-ending-work/" target="_blank">prematurely ending work</a>

The next pattern I want to discuss is executing conditional work. Sometimes, you have a stream and if some condition is met, you want to do some extra step. 

But, we do not have an `if-else` operator. And that makes sense. If we had such an operator, the entire `Observable` chain would be broken. Luckily there are a few ways to introduce conditional work inside of our `Observable` chains.

## Executing conditional work

Let's imagine that we have list of items in a webshop. In that list, we can check one or more items that we want to buy. When the user clicks on the 'buy' button, we first want to check if one of the items delays the delivery date by a huge amount. If so, we want to show a popup to notify the user. He can either decline and change his order or accept this.

In this use case, we have two `if-else`s. 
- If there is an order which has an item that pushes the delivery date we need to show a popup. Otherwise we don't.
- If the popup is shown, and the user accepts, we do a backend call to actually register the order. Otherwise we don't and the popup should close so the user can maybe remove that specific item.

Let's first take a look at what the code could look like.

```typescript
selectedItems$.pipe(
	'ifElse'(() => {
		// 
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
).subscribe(_ => // route to a different page here);
```

**Note:** `ifElse` is not a real operator 🙃

So we start with our `selectedItems$`. We then want to add our conditional logic. If we pass our conditional logic (either no delay or we showed the popup and the user accepted), we can perform the backend call. We can do this with the `switchMap` operator and using the `selectedItems`. At last we subscribe and we route to a different page if the call was successfull.

Now, how can we plug 	in in that conditional logic? First of all, let's think about how we can show a popup. We can have a function, that adds a popup to the DOM and returns an Observable with the result. The type signature for that function could look like this:

```typescript
showDialog(): Observable<boolean>;
```

So, inside of the `Observable` chain we had above, we want to conditionally execute the `Observable` that we get from the `showDialog` function so the dialog is shown. But, we only want to show this in case we have an item that delays the delivery date. 

Let's first implement what we need to do when there is no delivery delay (happy path 😄).

```typescript
selectedItems$.pipe(
	map(selectedItems => {
		// if none of the items has a late delivery
		if(!selectedItems.find(item => item.lateDelivery)) {
			return selectedItems;
		}
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
);
```

We add an if statement inside of a `map` operator. If the condition is not met, so there is no delayed delivery, we just return the `selectedItems` so these can be used in the `switchMap` operator to do the backend call. 

So far so good. BUT, how will we implement the else?

Inside of the `map` operator, we can implement the else case. In that else case, we want to show the dialog. This means, we are going to map to the result of the `showDialog` function.

```typescript
selectedItems$.pipe(
	map(selectedItems => {
		// if none of the items has a late delivery
		if(!selectedItems.find(item => item.lateDelivery)) {
			return selectedItems;
		} else {
			return showDialog();
		}
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
);
```

Of course, this introduces a problem inside of our code. The if statement will still work, but the else statement is posing a problem here. If the else is executed, the stream that is returned by the `map` operator (so before the `switchMap`) is of type `Observable<Observable<<boolean>>`. That's not what we need. The `switchMap` operator needs to be applied to an `Observable` with our selected items.

To fix this, we don't need to return the `dialog$` but:

- Subscribe to this stream (remember, `Observable`s are lazy and subscribing will trigger the dialog to be shown)
- Listen for the result of that `dialog$`, remember, the user can either accept or decline, and handle this properly

First step in fixing this, we can change the operator that we are using. If we change the `map` operator to a flattening operator such as `switchMap` or `concatMap`, we can return the `Observable` from the `showDialog` function and the operator will flatten it from an `Observable<Observable<boolean>>` to an `Observable<boolean>`.

Let's take a look at the code:

```typescript
selectedItems$.pipe(
	switchMap(selectedItems => {
		// if none of the items has a late delivery
		if(!selectedItems.find(item => item.lateDelivery)) {
			return of(selectedItems);
		} else {
			return showDialog();
		}
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
);
```

As you can see we changed from a `map` to a `switchMap`. Because we changed the operator, we have to make sure that every branch in the function we pass to `switchMap` returns an `Observable`. That's why we changed the `selectedItems` to `of(selectedItems)`. 

With this code, we are going to show the popup only if the condition is met. So we have a part of the conditional logic that we need. But, the overall code is not yet complete. The type of the `Observable` created by the first `switchMap` is `Observable<boolean|Array<Item>`. And that's not what we want. We don't want that boolean in there.

The last thing we need to do is check the value that we get back from the `dialog$`. This will return `true` if the user accepted the delay or `false` if the user denied. In that last case, we don't want to call the backend and this stream should stop.

Let's add this:

```typescript
selectedItems$.pipe(
	switchMap(selectedItems => {
		// if none of the items has a late delivery
		if(!selectedItems.find(item => item.lateDelivery)) {
			return of(selectedItems);
		} else {
			return showDialog().pipe(
				map(res => {
					if(res) {
						return of(selectedItems);
					} else {
						return never();
					}
				}),
			);
		}
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
);
```

Before returning the stream we get from the `showDialog` function, we are going to map its result to what we want. 

If the result was `true` we are going to return our `selectedItems`. But, since this is wrapped inside of a `switchMap` operator, we need to wrap this into an `Observable` using the static `of` operator. 

If the result was `false`, we can return the `never()` `Observable`. This is an `Observable` that will have no events whatsover. By doing this, the `Observable` chain is interrupted and `switchMap` that executes the backend call will never get an event and thus will not get executed (the one doing the backend call 🙃).

As as a last step, we want to make sure that we only take a single value. We start from the `selectedItems$` which can have potentially multiple values. For example when:

-  the user gets the popup, 
-  decides to cancel
-  selects or deselects a new item

the subscription would still be active. If the user selects a new item, the logic in our stream would fire immediately. We can fix this quite easily with the `take` operator.


```typescript
selectedItems$.pipe(
	take(1),
	switchMap(selectedItems => {
		// if none of the items has a late delivery
		if(!selectedItems.find(item => item.lateDelivery)) {
			return of(selectedItems);
		} else {
			return showDialog().pipe(
				map(res => {
					if(res) {
						return of(selectedItems);
					} else {
						return never();
					}
				}),
			);
		}
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
);
```
And that's it. This code does what we want! 🎉🎉

You can find a working (slightly contrived example) below. Click the buttons to trigger a delivery with or without delay. You can open the console to see a log statement being logged everytime the backend call would be executed.

<iframe style="width: 100%; height: 450px" src="https://stackblitz.com/edit/rxjs-hsqluy?embed=1&file=index.ts"></iframe>

## Using the `tap` operator

You can also hook into the `Observable` chain using the `tap` operator and maybe do some conditional work there. For example, to disable or enable a certain button:

```typescript
const selectedItems$ = ...

selectedItems$.pipe(
	tap(selectedItems => {
		if(selectedItems.length === 5) {
			this.newOrderBtnDisabled = true;
		} else {
			this.newOrderBtnDisabled = false;
		}
	})
);
```

I would argue however that it is better to create a new stream that contains the 'disable' state of that button. This can be achieved like this:

```typescript
const selectedItems$ = ...

const disabled$ = selectedItems$.pipe(
	map(selectedItems => selectedItems > 5)
);
```

This gives you exactly the same result and we do not need 'if-else' logic here.

## When to use this

Some examples where to use this pattern is:

- when a popup can decide if the action should be executed
- when creating a generic component that can show a spinner or not based on some configuration
- when creating a wizard and you only want to continue to the next step if the user current step is validated through a backend call








