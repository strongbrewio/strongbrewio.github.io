---
layout: post
cover: false
title: My favorite metaphor for hot vs cold observables
date:   2017-01-10
subclass: 'post'
categories: 'kwintenp'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS
cover: 'assets/images/cover/cover4.jpg'
---

A few weeks ago, I was at NG-BE (best conference of the year btw), where I was giving a workshop on RxJS and @ngrx/store in Angular 2 applications. In this course, we also explain hot vs cold observables.
When I try to explain new concepts to people, I either try to start from a known concept and build my way up to the new thing or use a metaphor. I've been looking for a similar approach to explain the hot vs cold observables concept but didn't found one myself.
At the conference, I started talking with <a href="https://www.linkedin.com/in/lander-verhack-a404a04b" target="_blank">Lander Verhack</a>. Lander gives trainings at U2U and he told me a great metaphor for this concept. I'll try to explain it below but all credits go to him for coming up with it.

### Watching a movie
When you want to watch a movie nowadays, you have a lot of different options. You can either go to the movies, rent out a dvd (do people still do this?) or use some service like Netflix.
Lets first think about what happens when you watch a movie via Netflix.

#### Netflix
First step in watching a movie on Netflix is looking one up in the catalogue. As soon as you have found one, you can decide to start watching it. A Netflix movie will, unless you've already watched a part, start from the beginning. It doesn't matter how many other people are watching the movie, you will always start at the beginning. If the movie doesn't live up to your expactations, you can decide to stop watching it.

It turns out that watching a movie on netflix is the perfect metaphor for a cold observable. Lets take a look at the common properties between these two:

* If no one is listening, the movie is not streamed. The movie stream is lazy, just like the cold observable.
* For every listener, a new stream is started. It's not because 5 people are already watching the movie somewhere else in the world, that your movie will not start from the beginning. The movie stream is unicast, just like a cold observable.

#### Going to the movies
Another way of watching a movie is by going to the movies. In contrast with Netflix, it's not you who decides when the movie will begin. It will begin at a predetermined time. When the movie starts, everyone who is present will see the same movie. If someone arrives too late, he will have missed the first part of the movie, but can follow everything afterwards the same way as the other people. If you decide to stop watching the movie because you don't like it, the other people can keep on watching it. If everybody decides to walk out because it's that bad, the movie will keep on playing.

Going to the movies is actually the perfect metaphor for a hot observable. Lets take a look at the common properties between the two:

* As soon as the movie starts, everybody who is watching it, sees the same thing. If a person arrives 5 minutes late, the movie will not restart just for him. He will, from his moment of arrival, see the same things. So watching a movie at the movies is the same as a hot observable because it is multicast.
* If the movie has already started, and you arrive 5 minutes later, you will have missed the first part. So just like a hot observable, the movie is not lazy.


### What about `publish().refCount()`

Aside from hot and cold observables, you can also create an observable that starts emitting values as soon as there is one subscriber. You can accomplish this with the `publish().refCount()` operator or its alias `share()`. I've seen people calling this observable 'warm', 'semi-hot' or 'semi-cold'.

#### Watching a movie with friends
You could think of this 'type' of observable as watching a movie with some (douchy) friends. Let's say you decide to watch a movie together at 6PM at your friends place. Due to car trouble, you'll be runnning 15 minutes late so you text your friend and ask him to wait.
Because he is so excited about the movie, he decides to start anyway. As expected, you arrive 15 minutes late and you keep on watching the movie, having missed the first part.

Let's see how this maps to the 'warm' observable we described above.

* Since your (douchy) friend decides to start watching the movie, you missed the first part. So, as soon as he starts watching the movie, the movie stream starts and all your friends that were on time, see the same values. Since you arrive too late, you'll miss the first part. This is exactly the same as our 'warm' observable.

### Conclusion
So, if you ever have to explain the concept yourself, or just as a way to better remember it, think of observables like this:

* Hot observable 		-> 		Movie at the movies
* Cold observable 		-> 		Movie with netflix
* 'Warm' observable 	-> 		Movie with friends
