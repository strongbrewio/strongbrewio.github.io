---
title: Safe HTTP calls with RxJS
published: true
author: brechtbilliet
description: How to make sure our ajax calls are being executed on bad connections
layout: post
navigation: True
date:   2018-10-16
tags: RxJS
subclass: 'post'
categories: 'brechtbilliet'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover3.jpg'
---

Hi there, since it's very busy lately this will probably be my one of my shortest articles ever.
Maybe that's a good thing, because now you dont have an excuse not to read it. It's short, compact
and maybe you will learn a thing or two.

## The problem

The article is all about making sure our HTTP calls don't die on bad connections, since strangely enough, **404 responses can kill your application when using RxJS**.

Remember that RxJS observables have 3 types of events right? 
- Next: passing in a new value into the observable
- Error: when an error occurs
- Complete: When the observable is completed

We should not forget that **an Error event will actually stop the observable**. It will cease to exist.

"That's not that bad, we'll just create a new one every time we want to fetch data",  you say?

Well when you are approaching your application the *reactive way*, this scenario might be problematic:

```typescript
// this observable contains the values
// of what the user is searching for
// over time
const searchTerm$: Observable<Result[]>;

// when the term receives a new value...
// go fetch some data
const results$ = searchTerm$.pipe(
    switchMap(term => fetchData(term))
)
// subscribe to the observable to start listening
results$.subscribe((response: Result[]) => {
    console.log('do something fancy');
})
```
This all works fine, untill the user gets a `500` error, or even a simple `404` error ...
If the user is having a bad connection which might result in a `404`, the observable will stop and the application will be broken. The user can search for results as much as he or she wants, the HTTP calls will never happen again.

## catchError

We could use the `catchError` operator that will basically catch the error for us, and return a brand new observable(containing the error).
That way we could actually show the user a decent message.
This might look something like this:

```typescript
const results$ = searchTerm$.pipe(
    switchMap(term => 
        fetchData(term).pipe(
            // return an observable with the error inside
            catchError(e => of(e))
        )
    )
)
results$.subscribe(
    (response: Result[] | HttpErrorResponse) => {
        if(response instanceof HttpErrorResponse){
            console.log('oh no:('));
            return;
        }
        console.log('do something fancy');
    });
)
```

Do note that the `catchError` operator is applied to the result observable that `fetchData()` returns, and not added as the second operator of the first pipe. 
From the moment an observable receives an error, it will die... That's why it's important to catch the error at the highest level. 

## retryWhen

Ok, great! The application won't break anymore, but what happens when the user is sitting on the train searching for awesome results, and he drives through a tunnel. The connection is gone for a few seconds and the user remains result-less.

We could fix that by telling RxJS to retry a few times

```typescript
const results$ = searchTerm$.pipe(
    switchMap(term => 
        fetchData(term).pipe(
            retryWhen(e$ => e$.pipe(
                // try again after 2 seconds
                delay(2000),
                // stop trying after 5 times
                take(4)
            )
            // still keep the observable alive if
            // the first 5 times fail
            catchError(e => of(e))
        )
    )
)
```

## Kicking it up a notch

We can do better than that. We can even use the [online](https://developer.mozilla.org/en-US/docs/Web/API/NavigatorOnLine/Online_and_offline_events) event from HTML5 to tell the browser to retry when the user regains internet connection. It's even shorter than before.

```typescript
const results$ = searchTerm$.pipe(
    switchMap(term => 
        fetchData(term).pipe(
            retryWhen(() => fromEvent(window, 'online'))
            // still keep the observable alive if
            // the server would return a different
            // HTTP error
            catchError(e => of(e))
        )
    )
)
```

## Conclusion

RxJS gives us great control over HTTP calls! If we know how error handling works it becomes a breeze to take our HTTP calls to the next level.

There, I told you it would be short, I hope you learned something though.

## Special thanks
[AmarildoKurtaj](https://twitter.com/AmarildoKurtaj) The last example was based on his idea