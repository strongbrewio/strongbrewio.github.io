---
title: A glitch in combineLatest (and how to fix it!)
published: true
author: kwintenp
description: Describes a glitch in using combineLatest and how we can fix this glitch
layout: post
navigation: True
date:   2018-04-25
tags: RxJS
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover4.jpg'
---

The `combineLatest` operator is probably one of my favorite ones, that I believe everyone should know. You should never try to learn all of them but `combineLatest`, to me, is definitely one of those ~15 you should probably understand.

**Note:** If you are unfamiliar with this operator, I suggest you check it out immediately <a href="http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#static-method-combineLatest" target="_blank">here</a>.

Even though it is one of the most well known operators, it can potentially introduce some weird behaviour. Let's try and find the weird behaviour and see how we can fix it.

## Identifying the problem

To do so, I created an application that visualises a number of pokemon based on a limit and offset parameter.

![example-app-screenshot](https://www.dropbox.com/s/u1autxtkhlp3nfk/Screenshot%202018-06-09%2013.44.16.png?raw=1)

Every time the limit or the offset changes, a backend call is triggered that will update the list of pokemon to be shown.

Let's take a look at how we can set up our stream to make this work. We will start by looking at the marble diagram.

![marble-diagram](https://vectr.com/tmp/f1jfpxhCHV/b8pN04Jr9.svg?width=1000&height=461.54&select=b8pN04Jr9page0)

We have a stream of the limit values and one for the offset values. We combine these streams using **the `combineLatest` operator to create a stream that will have a new value every time one of the source stream changes**. We then use the `switchMap` operator to fetch the data from the backend based on these values to get a `pokemon$`. Because we use `switchMap`, if a call is not finished yet, it will be cancelled when a new call is initiated by changing the limit or offset.

Code wise this looks like this:

```typescript
this.pokemon$ = combineLatest(
			limit$, 
			offset$, 
			(limit, offset) => ({limit, offset}))
      .pipe(
        switchMap(data => this.pokemonService.getPokemon(data.limit, data.offset)),
        map((response: {results: Pokemon[]}) => response.results),
      );
```

Here is the live example you can play with:

<iframe style="width: 100%; height: 450px" src="https://stackblitz.com/edit/angular-deqtkx?embed=1&file=src/app/app.component.ts"></iframe>

**Note:** If you open the devtools to the network tab and update the values pretty quick, you can see the calls being cancelled.

### I thought their was some weird behavior?

Everything seems to work fine right? So where is the hiccup?
Aside from the option to change the limit and offset values, there is also a 'reset' button. This button will set the values back to 5 and 0. 

```typescript
reset() {
    this.limitControl.setValue(5);
    this.offsetControl.setValue(0);
}
```

To see the hiccup, open your devtools on the network tab and check what happens when you click the button.

![gif-reset-clicked](https://www.dropbox.com/s/uzl8lprokyw99ew/reset-button-clicked.gif?raw=1)

Whenever the button is clicked, we can see that a call is initiated but immediately cancelled and a new call is started. That's a little strange no? 

### Explaining the behavior

Actually, this makes sense. In the description of the marble diagram above, there was a highlight: 

> 'the `combineLatest` operator to create a stream that will have a new value every time one of the source stream changes'.

By clicking the reset button, we updated both of our source streams by resetting both the limit and offset value at the same time. The effect of this action was that the stream created by the `combineLatest` operator fired twice, thus starting two backend requests, thus, cancelling one immediately because we used the `switchMap` operator. 

To make it even more clear, lets put it in steps.

- `combineLatest` holds the last values from all source streams (in the gif, the begin scenario was, limit = 8, offset = 2)
- the reset button is clicked
- limit is set to 5
- the `combineLatest` operator sees a new value coming in for limit and emits a new combination, limit = 5, offset = 2
- the `switchMap` operator gets these values and subscribes to the stream that triggers a backend call
- offset is set to 0
- the `combineLatest` operator sees a new value coming in for offset and emits a new combination, limit = 5, offset = 0
- the `switchMap` operator gets these values, unsubscribes (and thus cancels) the previous request and starts a new one

Something you might have not expected in this flow is that, whenever the limit is set, this change propagates to the `combineLatest` operator directly before changing the offset. 

**Note:** This is possible because RxJS does not have the notion of transactions. In a 'true' Functional Reactive Programming implementation, this would not be possible. Transactions would make sure there can be no simultaneous events. This is food for another post though :).

### How can we fix this?

If there was a way we could make sure that changes that happen in the same call stack (which is what is happening when clicking the reset button), are discarded in favor of the last change, we could fix our problem.

This means, that when the `combineLatest` operator emits two values in the same call stack, the last one is send through when the call stack is cleared.

To do this, we can leverage the `debounceTime` operator with a value of 0 directly after the `combineLatest`. This will make sure only the last value is passed through to the `switchMap` and this after the call stack has been cleared.

Let's put this in steps again to make it clear.

- `combineLatest` holds the last values from all source streams (in the gif, the begin scenario was, limit = 8, offset = 2)
- the reset button is clicked
- limit is set to 5
- the `combineLatest` operator sees a new value coming in for limit and emits a new combination, limit = 5, offset = 2
- the `debounceTime` operator sees a new value and (because of the 0) will wait until the call stack is cleared to pass it on
- offset is set to 0
- the `combineLatest` operator sees a new value coming in for offset and emits a new combination, limit = 5, offset = 0
- the `debounceTime` operator sees again a new value, will discard of the old one, and will wait for the stack to be cleared to pass it on
- the call stack is cleared
- the `debounceTime` operator sees no new value is given and will pass the combination, limit = 5, offset = 0, on
- the `switchMap` operator gets these values and subscribes to the stream that triggers a backend call


The updated code looks like this:

```typescript
this.pokemon$ = combineLatest(limit$, offset$, (limit, offset) => ({limit, offset}))
      .pipe(
        debounceTime(0),
        switchMap(data => this.pokemonService.getPokemon(data.limit, data.offset)),
        map((response: {results: Pokemon[]}) => response.results),
      );
``` 

You can play with the updated example here and see that the issue no longer happens.

<iframe style="width: 100%; height: 450px" src="https://stackblitz.com/edit/angular-gnlpt6?embed=1&file=src/main.ts"></iframe>

## Conclusion

When combining streams with the `combineLatest` operator, where the source streams might have new values within the same call stack, you might get unexpected behavior. You can fix this by adding a `debounceTime(0)` right after the `combineLatest`.
