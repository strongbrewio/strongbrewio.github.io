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

The `combineLatest` operator is probably one of my favorite operators that I believe everyone should know. You should never try to learn all of them but `combineLatest`, to me, is definitely one of those ~15 you should probably understand.

**Note:** If you are unfamiliar with this operator, I suggest you check it out immediately <a href"http://reactivex.io/rxjs/class/es6/Observable.js~Observable.html#static-method-combineLatest" target="_blank">here</a>.

Even though it is one of the most well known operators, it can potentially introduce some weird behaviour. Let's try and find the weird behaviour.

## Identifying the problem



https://stackblitz.com/edit/angular-deqtkx?file=src%2Fapp%2Fapp.module.ts