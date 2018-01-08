---
layout: post
cover: false
title: Using decorators and observables to implement retry
date:   2017-02-15
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS Typescript
---

Last week, I was at a meetup in Ghent where I was talking with <a href="https://twitter.com/stefanlapers" target="_blank">Stefan Lapers</a> about programming languages in general. We started talking about writing your backend using either Java or node.js. We agreed that node.js has massive potential but in a lot of situations companies choose Java because of it's maturity. If you're using a microservices based architecture for example, you can rely heavily on spring cloud which does a whole bunch of stuff for you so you can focus on functionality.
One element in spring cloud is hystrix. When you're doing a network call which is protected by hystrix, you can, by just adding an annotation, tell how many times you want to retry this if it fails and even provide a fallback if it fails entirely. 

When I was driving home later that night, I was thinking to myself that using observables and typescript decorators, it should be possible to implement something similar myself. The next morning I tried it out and about 30 minutes later I had a working version. Here it is.

## The example

In the example below, we have a service which fetches a number of Star Wars characters from the backend, at least it tries. It seems I kind of screwed up the implementation a little :).

```typescript
public getCharactersAndFail(): Observable<StarWarsCharacter[]> {
    return Observable.throw('Failing on purpose');
}
```
Instead of actually doing a backend call, I'm just returning an observable which will throw as soon as it's subscribed to. This is of course not to handy but for demonstration purposes, it's quite ideal.
Using the decorator I created myself, we can make this code a little more resillient (we'll dive into how this decorator is constructed later on). 

```typescript
@retry(3, [{name: 'Obi Wan', birth_year: '1234', gender: 'Male'}])
public getCharactersAndFail(): Observable<StarWarsCharacter[]> {
    return Observable.throw('Failing on purpose');
}
```

I've added the `retry` decorator. The first argument it takes tells the decorator the times it should retry the backend call. The second parameter is the fallback that will be used if the backend keeps on failing.

The following is a gif of what happens when you try to run this code:

![example-gif](https://www.dropbox.com/s/7natkxyj02o3xmd/Mar-08-2017%2019-32-51.gif?raw=1)

You can see that there are 3 tries before the method returns the fallback we defined. Pretty cool right.

A second example is if you click the button below the first example. This will actually try to do the real request.
The code for this backend call looks like this:

```typescript
@retry(3, [{name: 'Obi Wan', birth_year: '1234', gender: 'Male'}])
public getCharacters(): Observable<StarWarsCharacter[]> {
    return this.http.get('https://swapi.co/api/people/')
      .map((response: Response) => response.json().results);
}
```

For the retry example here to work, you have to go offline first, which will cause the request to fail. If you then try to run this code, you can see in the network tab it actually tries to do 3 requests, which fail because we are offline. After three attempts, the fallback is returned.

![example-gif](https://www.dropbox.com/s/494jhrwvtdelo09/Mar-08-2017%2019-35-31.gif?raw=1)

You can find the live working example <a href="http://blog-kwintenp-examples.surge.sh/home/retry" target="_blank">here</a>. Open up your console to see the number of tries and inspect the network tab with the second example.

## The implementation

To implement this, there are two important things we need to do. First is implementing the logic to retry the call if it failed. We can levarage streams for this. Next we need to extract that logic into a typescript decorator.

### Observable composition

Retrying a subscription after it has failed is fairly easy using RxJS. We can use the `retryWhen` operator for this. Let's look at some code.

```typescript
private starWarsService: StarWarsService;

characters$: Observable<StarWarsCharacter> = 
	// we call the starWarsService to get the characters
	this.starWarsService
		.getCharacters()
		// we use the retryWhen operator to retry a number of times
		.retryWhen((errors: Observable<any>) => {
		  // we use the scan operator to count the number of tries
          return errors.scan((errorCount, err) => {
            console.log('Try ' + (errorCount + 1));
            if (errorCount >= 3) {
              throw err;
            }
            return errorCount + 1;
          }, 0).delay(1000);
        })
        // we catch the error if it keeps on failing and return
        // the fallback
        .catch(() => Observable.of(fallback));
```
I'm not going to go into the details of the RxJS implementation, that whould require a totally separate post.
What this code does however, is create an observable that, once subscribed to, will try to execute the backend call. If it fails, which it will in our case, it will re-execute it 2 more times with a single second delay in between. If it still fails, it will return the fallback we can define.

That's the exact logic we want our decorator to do. So let's see how we can extract this logic in the decorator.

### Creating a decorator

There are different types of decorators. We can put a decorator on a class, method, property or accessor method. In our case, we are going to use the method decorator. A method decorator is in fact nothing more than a function that gets called at runtime. You can find more information on what decorators are and how to use them <a href="https://www.typescriptlang.org/docs/handbook/decorators.html#method-decorators" target="_blank">here</a>

Using a method decorator, you can replace, observe or modify the method definition. What we are going to do is really easy. Lets take a look at the image below.

![method decorator](https://www.dropbox.com/s/o3xef1gl9f4jlmd/Screenshot%202017-03-05%2014.14.48.png?raw=1)

At runtime, when the method we are decorating gets called, the decorator will be called first (1). We are going to call the actual function (2) which is going to return an observable in our case (3). Before returning the observable to the caller of our decorated function (5), we are going to augment the observable with our retry logic (4) and return this new observable.

Lets take a look at the code:

```typescript
// defining the decorator is the same as the defining a
// function with the same name
export function retry(times = 3, fallback: any) {
  return (target, key, descriptor) => {
    // the descriptor holds a reference to the actual method
    // we are decorating
    const originalMethod = descriptor.value;
    // we replace the old function with a new function
    descriptor.value = function () {
      // call the original method and
      // augment the resulting observable
      // with the retry and fallback mechanism
      // we defined above
      return originalMethod.apply(this)
        .retryWhen((errors) => {
          return errors.scan((errorCount, err) => {
            console.log('Try ' + (errorCount + 1));
            if (errorCount >= times - 1) {
              throw err;
            }
            return errorCount + 1;
          }, 0).delay(1000);
        })
        .catch(() => Observable.of(fallback));
    };
    // return edited descriptor as opposed to
    // overwriting the descriptor
    return descriptor;
  };
}
```
In the decorator we take the original function. We replace this function with a new one that executes the original one and augments the resulting observable with our retry logic.

Once you've understood the syntax of the method decorator, this implementation is pretty straight forward. It now allows us to add the decorator on top of every function that returns an observable like this:

```typescript
@retry(3, [{name: 'Obi Wan', birth_year: '1234', gender: 'Male'}])
public getCharactersAndFail(): Observable<StarWarsCharacter[]> {
    return Observable.throw('Failing on purpose');
}
```

## Conclusion

Using the ease of composing of observables and decorators in typescript, we were able to create something really fast which can be reused throughout our entire application. It shouldn't stop with a retry decorator. We could apply this principle to a whole bunch of RxJS related issues like catching errors for example.
