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

A lot of people using RxJS believe that this is a form of Functional Reactive Programming (FRP). While this is probably not a wrong statement, it is also not entirely correct. To be a true FRP implementation, RxJS should have the option to handle simultaneous events.

- not truly simultaneous 
- Transaction mgmt
- debounceTime
- 