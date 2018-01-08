---
layout: post
cover: false
title: Introducing Rx devtools
date:   2017-08-04
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS
---

Ever since I first started using RxJS up until this very day, it has become my absolute favorite way of coding. I cannot believe to work in a world without observables anymore nor can I understand on how I was able to write code before. I have started sharing the knowlegde I had through blogposts, workshops and coaching. 

One thing that always comes up while working with observables is the very high learning curve. It's really hard to grasp the concept when you're just starting. One of the reasons for this is that it's really hard to visualize and debug the observables in your application. 
With this in the back of my mind, I started wondering how that could be fixed. If there only was a way to clearly see the data flowing in the streams of your application in realtime. That's how the idea for Rx Devtools was born.

## Introducing Rx Devtools	

After first playing with the idea, I decided to create a small POC. This POC has grown into a chrome extension that, as of today, can be used to visualise streams realtime! Take a look at the demo below (it's a youtube video, pls click :)):

[![Rx devtools teaser](https://img.youtube.com/vi/stWGClDE_Gk/0.jpg)](https://youtu.be/stWGClDE_Gk)

On the left you can see the code we are debugging at the moment. Notice the `debug` operators on every observable. Here you can pass a name to track the streams. 
On the right side you can see the plugin in action. Left, we have a list with one entry per observable we are debugging. When clicked on one of them, you can see the actual marble diagrams with all of the operators. You can click on a marble to inspect the value it had at that moment in time. This way, you can not only see the value of every event being passed to the observable chain, but also see the moment in time they were produced and push down this chain.

If you for example have a combineLatest which doesn't seem to fire, there will probably be one source observable that is not producing a value. With the plugin, this is visualised in seconds!

For more information on the plugin, how to install, how it works and how to use it, I would like to point you to the <a href="https://github.com/kwintenp/rx-devtools" target="_blank">Github</a> page.


### What's next
The plugin as it exists today can definitely be used. It is however far from finished and still in an alpha phase. Over the next few weeks, I'll try to add as many features asap. If you have any ideas for features you want to see added, feel free to create feature requests through Github issues. 
If you find any bugs, which I'm certain you will, please report them in the form of Github issues. I will try to tackle them asap.

Happy debugging!

