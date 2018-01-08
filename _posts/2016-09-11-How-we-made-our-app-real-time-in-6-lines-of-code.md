---
layout: post
title: How we made our app real-time in 6 lines of code
published: true
description: Kwinten Pisman and me were working on a workshop this weekend with the focus on Reactive applications with Angular, RXJS and @ngrx. Something that can't miss in a reactive workshop are real-time updates. The application we are trying to make real-time is the winecellar app (you can register an account here if you want to test it).
author: brechtbilliet
comments: true
date: 2016-09-11
subclass: 'post'
categories: 'casper'
disqus: true
tags: Angular
---
[Kwinten Pisman](https://blog.kwintenp.com/) and me were working on a workshop this weekend with the focus on Reactive applications with [Angular 2](http://angular.io), [RXJS](https://github.com/ReactiveX/rxjs) and [@ngrx](https://github.com/ngrx). Something that can't miss in a reactive workshop are real-time updates. The application we are trying to make real-time is the [winecellar](http://winecellar.surge.sh) app (you can register an account here if you want to test it).

To make this application real-time we changed some code in the node.js backend, but that's out of scope for this post. 
The cool thing is, that we only needed **6 lines of code** to make the frontend completely real-time.

Here's a small demo. Both computers are signed in with the same account. At the left screen, wines are being added and removed, and in the right screen you'll see the changes happening real-time.
![Winecellar app](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/realtimein6lines/realtimewinecellar.gif)



## The winecellar app

The winecellar application is a simple application with only a few features, but we overengineered it on purpose to make it work for largescale applications.
You can find the open-source code of the winecellar project here: [frontend, (realtime branch)](https://github.com/brechtbilliet/winecellar) and [backend](https://github.com/brechtbilliet/WineCellarBackend). Beware, the backend might be a little quick and dirty ;)

### Technology stack

<ul>
<li>Angular 2</li>
<li>RXJS</li>
<li>@ngrx/store as redux implementation</li>
<li>Typescript</li>
<li>We will use socket.io to make the real-time connection with the backend</li>
</ul>

### features

<ul>
<li>Authentication</li>
<li>Add, update, remove wines</li>
<li>Searching in a public wine database (wine.com)</li>
<li>Filtering wines</li>
<li>Update the stock of wines</li>
<li>Setting ratings for wines</li>
</ul>

![Winecellar app](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/realtimein6lines/winecellar.png)

## What part of the application should be real-time?

<ul>
<li>All the actions that will update the content in the database eventually should push notifications to all clients except the one sending them.</li>
<li>Even the filter should be real-time. (Eg. when filtering on "Chateau pomerol", and somebody adds that wine, we want to update our filtered results real-time)</li>
</ul>

But why did we made something like that real-time? Actually, **just because we can**! It doesn't really make sense that a user is logged in twice right? We did it for the purpose of the workshop.

## How did we manage to make it real-time?

First, let me give you a little bit of information about how the winecellar really works.

Like said before we use @ngrx/store as our redux implementation. It is a redux library that we use to maintain the state of our application. Basically, you can see it as a client-side store of all our wines and other state. We use that particular store as a single-source-of-truth. We will send actions towards that store which will update the state with pure functions called reducers.

In the following scheme you can see the unidirectional dataflow of redux: The view sends actions to the store which will update the state and then update the view.
![Redux](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/realtimein6lines/redux.png)

Let's say that we want to add a wine. when we add a wine, we send a XHR request to the backend and send an action to the store, like you can see in the snippet below. That is already how to wine application works

```typescript
add(wine: Wine): void {
    this.http.post(`${API_URL}/wines`, wine, this.authorizedHttpOptions())
        .map((res: Response) => res.json())
    	.subscribe(resp =>
    		this.store.dispatch({type: DATA_WINES_ADD, payload: {wine:resp}})
		);
}
```

This doesn't have anything to do with real-time, right?! You are right, it doesn't... But what if our backend can send Redux actions as well? If the backend can push redux actions to the frontend, then we can make it real-time with very little effort.

For every REST call where something in the database gets updated, we can send a redux action to all the clients which are logged in with the same username (except for ourselves)

**These are the 6 lines of code, that we need to make our frontend 100% real-time.**

```typescript
import * as io from "socket.io-client";
// connect with socket.io and listen to redux actions
connect(): void {
    this.store.select(state => state.data.authentication.jwtToken)
    	.take(1).subscribe((token: string) => {
	        let socket = io(BACKEND, {query: "jwttoken=" + token});
	        socket.on("UPDATE_REDUX", action => this.store.dispatch(action));
    });
}
```


This is a simplified example of what happens on the backend when we add a wine, for instance.

```typescript
@Post("/")
public post(@Req()req: Request, @Res() res: Response): void {
    let userId = handleAuth(req, res);
    new Wine(req.body).save((error, response) => {
        ...
        // emit a socket.io event to the client
        // which contains the redux action
        this.pushToClient(userId, req, {type: DATA_WINES_ADD, payload: {wine: response}});
        // return the new user
        res.send(response);
    });
}
```

## Check the devtools
In The devtools below you'll see the actions being dispatched during the lifecycle of the application. These actions can be triggered manually or real-time. Be sure to check them out at [https://github.com/ngrx/store-devtools](https://github.com/ngrx/store-devtools)
![Winecellar devtools](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/realtimein6lines/winecellar_devtools.png)

## Conclusion

The data that is managed by redux can be easily made real-time by making the backend dispatch redux actions to the frontend. 
That way, we can make our application real-time in matter of minutes.
If you want to learn more about these technologies: Be sure to checkout our workshop at [ngbe](http://ng-be.org).