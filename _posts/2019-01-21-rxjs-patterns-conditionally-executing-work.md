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

This post is part of a series of blogpost on different RxJS patterns that I use quite often. Here are the previous ones:

- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">Restarting work</a>
- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">mapping a function to a stream</a>

The next pattern I want to discuss is executing conditional work. Sometimes, you have a stream and if some condition is met, you want to do some extra step. 

But, we do not have an `if-else` operator. And that makes sense. If we had such an operator, the entire `Observable` chain would be broken. Luckily there are a few ways to introduce conditional work inside of our `Observable` chains.

## Executing conditional work

Let's imagine that we have list of items in a webshop. In that list, we can check one or more items that we want to buy. When the user clicks on the 'buy' button, we first want to check if one of the items delays the delivery date by a huge amount. If so, we want to show a popup to notify the user. He can either decline and change his order or accept this.

In this use case, we have two `if-else`s. 
- If there is an order which has an item that pushes the delivery date we need to show a popup. Otherwise we don't.
- If the popup is shown, and the user accepts, we do a backend call to actually register the order. Otherwise we don't and the popup closes.

Let's first take a look at what the code could look like.

```typescript
selectedItems$.pipe(
	'ifElse'(() => {
		// 
	}),
	switchMap(selectedItems => this.service.buy(selectedItems))
).subscribe(_ => // route to a different page here);
```

**Note:** `ifElse` is not a real operator :)

So we start with our `selectedItems$`. We then want to add our conditional logic. If we pass our conditional logic, we can perform the backend call, we use `switchMap` to do this. At last we subscribe and we route to a different page if the call was successfull.

Now, how can we plug 	in in that conditional logic? First of all, let's think about how we can show a popup. We can have a function, that adds a popup to the DOM and returns an Observable with the result. The type signature for that function could look like this:

```typescript
showDialog(): Observable<boolean>;
```

So, inside of the `Observable` chain we had above, we want to conditionally map this to the dialog we want to show. 

But, we only want to show this in case we have an item that delays the delivery date. Let's add that piece of logic first.

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

We add an if statement inside of a `map` operator. If the condition is not met, we just return the `selectedItems`. 

So far so good. BUT, how will we implement the else?

We can use the same map operator, but in the else case, we want to map to the `Observable` we get from the `showDialog` function.

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

Inside the else statement, we return the `dialog$`. This means that, for the else statement, we get an `Observable<Observable<boolean>>` (remember that our `showDialog` function returns an `Observable<boolean>`. 

That's not what we want. We want to:

- Subscribe to this stream (remember, `Observable`s are lazy and subscribing will trigger the dialog to be shown)
- We want the result of the `Observable` and not the `Observable` itself.

To fix this, we can change the operator that we are using. If we change the `map` operator to a flattening operator such as `switchMap` or `concatMap`, we can return the `Observable` from the `showDialog` function and the operator will subscribe to the `Observable` for us and return the result!

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

Because we changed the operator, we have to make sure that every branch in the function we pass to `switchMap` must return an `Observable`. That's why we changed the `selectedItems` to `of(selectedItems)`. 

With this code, we are going to show the popup only if the condition is met. So we have a part of the conditional logic that we need. But, the overall code is not yet complete. The type of this `Observable` is `Observable<boolean|Array<Item>`. And that's not what we want. We don't want that boolean in there.

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

Before returning the stream we get from the `showDialog` function, we are going to map its result to what we want. If the result was `true` we are going to return our `selectedItems`. But, since this is wrapped inside of a `switchMap` operator, we need to wrap this into an `Observable` using the static `of` operator. If the result was `false`, we can return the `never()` `Observable`. This is an `Observable` that will immediately complete and has no next event. By doing this, the `Observable` chain is interrupted and the next `switchMap` statement will not get executed (the one doing the backend call :)).

And that's it. This code does what we want.


https://stackblitz.com/edit/rxjs-hsqluy?embed=1&file=index.ts

## Using the `tap` operator

You can also hook into the `Observable` chain using the `tap` operator and maybe do some conditional work there. For example:

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

// stream van items bevat 1 die de levering vertraagt (popup tonen om het te vragen)





