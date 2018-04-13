layout: post
title: Redux (@ngrx/store) best practices
published: false
description: This article is all about what to do and what not to do when using Redux (@ngrx/store)
author: brechtbilliet
comments: true
date: 2016-12-21
subclass: 'post'
categories: 'brechtbilliet'
disqus: true
tags: Redux, @ngrx, Angular
cover: 'assets/images/cover/cover2.jpg'
---

[@ngrx/store](https://github.com/ngrx/platform/blob/master/docs/store/README.md) is a library that tries to solve the problems of statemanagement through the principles of [Redux](https://redux.js.org/). The difference between Redux and @ngrx/store is that @ngrx/store is written specifically for Angular and it embraces the use of Observables from [RxJS](http://reactivex.io/rxjs/).
The combination of redux principles and RxJS can be very powerful when it comes to writing reactive applications.
Since a lot of Angular projects use @ngrx/store, it might be a good idea to write down some best-practices.

Note: The best-practices and opinions described in this article are strictly personal. Best practices are almost always a matter of opinion. Nevertheless, we (StrongBrew) are using these best practices at all our customers on a daily basis and they certainly work for us.
From now on @ngrx/store will be reffered to as Redux in this article.

## To Redux or not to Redux?

The first question that we might want to ask ourselves is do we really need Redux in our application.
It is a best practice to only use it when your application demands it.
I have written [an article](https://blog.strongbrew.io/do-we-really-need-redux) that tackles this question separately.

## What do we put in the store?

We shouldn't put things in the store just because we can. We have to think about what state needs to be in the store and why.
State that is being shared between components can sometimes be kept in the parent component for instance. When state needs to be shared between different root components (rendered inside a router-outlet) we might want to keep that state into the store.

When we need to remember a value when navigating through the application we could put that in the store as well. An example here could be: Remembering if a sidebar was collapsed or not, so when we navigate back to the page with the sidebar, it would still be collapsed.

## Don't forget about router params

A common mistake is putting things inside the store that could easily be added in the url.
The benifit of keeping state in the url is:

- We can use the browser its navigation buttons
- We can bookmark the url
- We can share that url with other people

Example: We have a map of a city with houses. On every house that we click we have to zoom to that house and the information of that house could be shown in a sidebar
![map](/assets/images/posts/redux-best-practices/map.png)
