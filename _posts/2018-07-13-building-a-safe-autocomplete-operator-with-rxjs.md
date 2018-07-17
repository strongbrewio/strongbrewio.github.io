---
layout: post
title: Building a safe autocomplete operator in RxJS
date:   2018-07-13
subclass: 'post'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: brechtbilliet
tags: RxJS
cover: 'assets/images/cover/cover6.jpg'
---

A well-known use case of RxJS is creating a simple autocomplete search in only a few lines of code.
This article is not just about creating our own autocomplete operator, we will create an autocomplete operator that is completely safe (we will cover the perception of safe later in this article).

When creating a standard autocomplete with RxJS we most likely implement a `debounceTime` operator to throttle the text that the user is typing into the autocomplete and we use a `switchMap` operator instead of a `mergeMap` operator to abort previous calls.

The implementation of this autocomplete might look like this:

```typescript
const term$ = new BehaviorSubject('');
const results$ = term$
    .pipe(
        // wait until the user stops typing for a second
        debounceTime(1000),
        // higher order observable, abort previous
        // call if still busy
        switchMap(term => getAutocompleteSuggestions(term))
    )
```

## The problem

When the user stops typing for 1 second, the browser will create a new XHR call. From that moment on, when the user types again and a previous XHR call is still busy, the browser will abort that XHR call to avoid racing conditions, and create a new XHR call. This is due to the `switchMap` operator since it will unsubscribe from the previous observable.

Although that's great, there is one problem though. What if the user starts typing again when an XHR call is still busy? 
Since we have implemented a `debounceTime` operator, the call will not be aborted until one second has passed.
In that period of time new results might be returned and showed to the user, which might not be what we want since **they are not relevant anymore**.

What we want is that the XHR call gets aborted from the moment the user starts typing again, we don't want to wait for a second.
This sounds like an ideal scenario to write our custom operator.
But, before creating our own operator, let's just combine some operators to get the job done.

A possible solution for this problem is using a `takeUntil` operator on the observable that will trigger the XHR call.
The `takeUntil` operator will complete the observable as soon as it gets a value. We want to complete (and therefore abort) the observable when the user types again. With that knowledge we can write something like this:

```typescript
const term$ = new BehaviorSubject('');
const results$ = term$
    .pipe(
        debounceTime(1000),
        switchMap(term => 
            getAutocompleteSuggestions(term)
                .pipe(
                    takeUntil(term$) // this still won't work
                )
            )
        )
    )
```

Sadly, this does not work yet.

Our `term$` observable is a `BehaviorSubject` for two reasons:
- We want to pass an initial value to the subject
- A `BehaviorSubject` is a `ReplaySubject(1)` behind the scenes that keeps track of the last value. This is important if we want to subscribe to that observable in a later stage (which is kinda what we do with the `takeUntil` operator).

Because the `term$` observable is keeping track of the last value, the `takeUntil` operator will always have a value, resulting in the fact that every XHR call gets aborted immediately. This is not what we want.
We want to skip one value of the `term$` observable every time.
We can achieve that with the `skip` operator as shown in the following example:

```typescript
const term$ = new BehaviorSubject('');
const results$ = term$
    .pipe(
        debounceTime(1000),
        switchMap(term => 
            getAutocompleteSuggestions(term)
                .pipe(
                    takeUntil(
                        //skip 1 value
                        term$.pipe(skip(1))
                    )
                        
                )
            )
        )
    )
```

Now the following scenario works:
- User types 'l'
- Application waits for a second
- Application creates an XHR call
- User types 'lu'
- Even though the XHR call wasn't finished yet it gets aborted immediately (it doesn't wait for a second anymore to abort that XHR call)

Because of that the user never gets irrelevant data on its screen.

## Extracting the logic into a custom operator

We don't want to write this logic every time, so let's extract this logic into a custom written operator.

Turns out that creating custom operators is super easy. An operator is just a function that returns a function that gets the source observable.

```typescript
const autocomplete = (/* additional parameters */) => 
    (source$) => source$.pipe(/* do stuff */ )
```

We can pass the `time` and `selector` function as parameters and use the operators we have written to create our own custom operator.
The operator looks like this:

```typescript
const autocomplete = (time, selector) => (source$) =>
  source$.pipe(
    debounceTime(time),
    switchMap((...args: any[]) => selector(...args)
        .pipe(
            takeUntil(
                source$
                    .pipe(
                        skip(1)
                    )
            )
        )
    )
  )
```

Using our operator is super easy:

```typescript
const term$ = new BehaviorSubject('');
const results$ = term$
    .pipe(
        autocomplete(1000, term => getAutocompleteSuggestions(term))
    )
```

You can find the sourcecode on stackblitz.
<iframe src="https://stackblitz.com/edit/safe-switchmap?embed=1&file=src/app/app.component.ts" style="width: 100%; height: 500px"></iframe>

## Conclusion

The combination of `debounceTime` and `switchMap` don't always cover everything. Showing irrelevant data to our users might not be what we want and creating our own operators is super easy! I hope you enjoyed the article.

## Special thanks

A special thanks for the awesome reviewers:

- Nicholas Jamieson [@ncjamieson](https://twitter.com/ncjamieson)
- Philippe Martin [@feloy2](https://twitter.com/feloy2)
- Jan-Niklas Wortmann [@niklas_wortmann](https://twitter.com/niklas_wortmann)
- Maarten Tibau [@maartentibau](https://twitter.com/maartentibau)
- Kwinten Pisman [@kwintenp](https://twitter.com/kwintenp)
