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

The `combineLatest` operator is probably one of my favorite operators, that I believe everyone should know. You should never try to learn all of them but `combineLatest`, to me, is definitely one of those ~15 you should probably understand.

**Note:** If you are unfamiliar with this operator, I suggest you check it out immediately <a href"http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#static-method-combineLatest" target="_blank">here</a>.

Even though it is one of the most well known operators, it can potentially introduce some weird behaviour. Let's try and find the weird behaviour.

## Identifying the problem

To do so, I created an application that visualises a number of pokemon based on a limit and offset parameter.

![example-app-screenshot](https://www.dropbox.com/s/u1autxtkhlp3nfk/Screenshot%202018-06-09%2013.44.16.png?raw=1)

Every time the limit or the offset changes, a backend call is triggered that will update the list of pokemon to be shown.

Let's take a look at how we can set up our stream to make this work. We will start by looking at the marble diagram.

![marble-diagram](https://vectr.com/tmp/f1jfpxhCHV/b8pN04Jr9.svg?width=1000&height=1000&select=b8pN04Jr9page0)




https://stackblitz.com/edit/angular-deqtkx?file=src%2Fapp%2Fapp.module.ts