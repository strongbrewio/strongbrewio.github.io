---
layout: post
title: The SIP principle
date:   2018-04-27
subclass: 'post'
categories: 'kwintenp'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: brechtbilliet
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---
A few months ago I have written [RxJS best practices in Angular](https://blog.strongbrew.io/rxjs-best-practices-in-angular/) and a while before that  [Thinking reactively in Angular and RxJS](https://blog.strongbrew.io/thinking-reactively-in-angular-and-rxjs/). 
Both of these articles are trying to make the mindswitch towards reactive programming easier.

However, sometimes we love to have structured opinionated ways of tackling stuff, especially when things become complex. We like a roadmap of some kind, something to fall back on, something to guide us through these complex scenarios.

While writing RxJS code for small pragmatic solutions is super easy, it might become complex when combining streams or doing other advanced stuff. Especially when we make the mindswitch completely towards reactive programming things tend to become complex in the beginning. 

We as StrongBrew are huge fans of reactive programming and
we use our reactive mindset in Angular every single day.
In this article I would love to teach you a principle that helps us to tackle very complex RxJS situations.

## The situation

We are going to build an application to search for starships in the [swapi api](https://swapi.co). The application counts a few features:
- It has to start loading data on page load
- The user can search for starships by name
- The user can show starships by model
- The user can load starships by a random model
- There is a loading indicator that needs to be shown when the data is being loaded
- Previous XHR calls should be canceled
- We want to debounce the search for 200 ms
- We want to client side filter the results by the number of passengers allowed on the ship

There is a lot of asynchronous logic going on here, when we would design the implementation in an imperative way, it would be pretty hard to keep it simple. This application is easily written with the use of observables.

## The SIP principle

## Special thanks
[Jurgen van de Moere](https://twitter.com/jvandemo) for helping us with the acronym.