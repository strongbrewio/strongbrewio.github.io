---
layout: post
cover: false
title: Code formatting and RxJS
date:   2017-09-08
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS
---

One of the main benefits of RxJS to me is that it provides code that is 'readable'. It provides us with a declarative programming approach where implementation details are hidden away. We are just describing what needs to be done, not how it should be done. This makes code you look at easy to understand.

One problem that I face regularly when looking at other peoples code is that the code formatting or the way the (RxJS) code is written, takes away part of the benefit of writing your code this way. This post is a small summary on how I like to format my code to keep the readability benefits to a maximum.

### One operator per line

One thing I see all the time is something like this:

```typescript
Rx.Observable.interval(1000).map(x => x*2).filter(x => x%2 === 0)
	.mergeMap(x => someBackendCall(x)).map(res => res.json()).subscribe();
```

Here we can see a simple stream. Looking at what is does is a little more difficult however because of the outlining of the operators. If you format the code like this, it makes it so much easier:

```typescript
Rx.Observable.interval(1000)
	.map(x => x*2)
	.filter(x => x%2 === 0)
	.mergeMap(x => someBackendCall(x))
	.map(res => res.json())
	.subscribe();
```

By putting every operator on a new line, it is so much easier to see what's happening.

### Using nested functions for functions longer than one line

A lot of the RxJS operators will accept functions as parameters. These functions can influence the code formatting and impair the readability. Let's take a look at an example:

```typescript 
// A data$ stream
private data$: Observable<Array<Data>>;

// A function that uses this data$ 
doSomething() {
    this.data$
    	.take(1)
        .map((data) => {
            if (data && data.length > 0) {
                	return data.forEach(datum => {
                    datum.active = false;
                	});
            } else {
            		return [];
            }
        })
        .mergeMap((data) => {
            data.forEach(datum => {
                this.whateverService.update(data);
            });
        })
        .subscribe();
}
```

We have a function `doSomething()` that, when called, will use the `data$` stream as a source and will perform a mapping of the data array events inside of this stream and then will perform a backend call for every element inside this array. 
If you were able to detect this immediately, my hat off to you. To me however, this looks pretty bad. Let's take a look at how we could make this better:

```typescript
// A data$ stream
private data$: Observable<Array<Data>>;

// A function that uses this data$ 
doSomething() {
    const mapAllTheElementsActiveFlagToFalse = (data) => {
        if (data && data.length > 0) {
            return data.forEach(datum => {
                datum.active = false;
            });
        } else {
            return [];
        }
    };

    const callTheWhateverServiceForEveryElement = (data) => {
        data.forEach(datum => {
            this.whateverService.update(data);
        });
    }; 

    this.data$
    	.take(1)
        .map(mapAllTheElementsActiveFlagToFalse)
        .mergeMap(callTheWhateverServiceForEveryElement)
        .subscribe();
}
```

I updated the code so that all the functions passed to the operators are first created as nested functions. This might feel a little weird at first, creating nested functions, but if we look at the last few lines of code, these have become so much cleaner. If you know what the operators of RxJS do, you can actually read what is happening (I must admit, naming these functions might not be my strongest feat :)). You only have to look at the last lines of this function. The implementation details of the nested functions is irrelevant (remember, declarative is easier to read).
I find this approach really helpful and tend to use it a lot, especially for functions that are longer than a single line. 

### Avoid using nested observables

RxJS provides us with a lot of operators which you can do a whole range of stuff with. One of them is combining different streams. One thing I sometimes see is this:

```typescript
private data$: Observable<Array<Data>>;
private data2$: Observable<Data2>;

result$: Observable<any>;

doSomething() {
	 // we combine the data$ and data2$ with combineLatest
    this.result$ = Observable.combineLatest(
        this.data$
            .map(data => data.length),
        this.data2$
            .mergeMap(val => this.whateverService.call(val)),
        (val1, val2) => {
            // handle the values here   
        }
    );
}
```

We create an observable by using the `combineLatest` operator. But before we do so, the `data$` and `data2$` streams are transformed. You could say we are working with nested streams. And even though, the operators are aligned perfectly and there are no functions that are longer than one line, it still feels weird. Let's see how we might be able to make it better:

```typescript
private data$: Observable<Array<Data>>;
private data2$: Observable<Data2>;

result$: Observable<any>;

doSomething() {
    const dataLength$ = this.data$
        .map(data => data.length); 
    const whateverData$ = this.data2$
        .mergeMap(val => this.whateverService.call(val));

	 // we combine the data$ and data2$ with combineLatest
    this.result$ = Observable.combineLatest(
        dataLength$,
        whateverData$,
        (val1, val2) => {
            // handle the values here   
        }
    );
}
```

This time, I extracted the nested observables and matched them to local variables first. I then use these newly created local variables to create a new stream using the `combineLatest` operator. 
By extracting nested observables to a separate variable and naming this new observable properly, the code is easier to understand.

**Conclusion** 

Try to keep the following in mind when writing RxJS code:

1. Put every operator on a new line.
2. Extract functions longer than a line to a nested function.
3. Try to avoid working with nested observables.













