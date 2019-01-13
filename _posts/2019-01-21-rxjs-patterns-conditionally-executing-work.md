---
layout: post
title: RxJS patterns - Conditionally executing work
published: true
description: RxJS patterns conditionally executing work
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---

This post is part of a series of blogpost on different RxJS patterns that I use quite often. Here are the previous ones:

- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">Restarting work</a>
- <a href="https://blog.strongbrew.io/rxjs-patterns-restarting-work/" target="_blank">mapping a function to a stream</a>

The next pattern I want to discuss is executing conditional work. Sometimes, you have a stream and in if some condition is met, you want to do some extra step. 

But, we do not have an `if-else` operator. And that makes sense. If we had such an operator, the entire `Observable` chain would be broken. Luckily there is a quite easy way to introduce conditional work inside of our `Observable` chains.

## Executing conditional work

Let's imagine that we have list of items in a webshop. In that list, we can check one or more items that we want to buy. But for some weird reason (and due my lack of creativity), you can only add 5 items to your basket. 

We have a grid component which exposes a stream with all of the selected items. 



// stream van items bevat 1 die de levering vertraagt (popup tonen om het te vragen)





