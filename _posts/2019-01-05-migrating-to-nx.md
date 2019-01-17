---
layout: post
title: Migrating to Nx
published: true
description: Migrating multiple apps to an nx workspace
author: kwintenp
comments: true
date: 2019-01-05
subclass: 'post'
categories: 'nx'
disqus: true
tags: Angular nx architecture
cover: 'assets/images/cover/cover2.jpg'
---

I have been using a monorepo with Nx for a while now (~1 year, an almost infinite period in javascript land 😂). And I must say, this has been a real eye opening experience. I've gotten to the point that everyone that is remotely interested into talking to me is going to get a pitch on why monorepos are the best and how Nx can help us in achieving this. I even gave a talk about this very topic at NG-BE which you can find <a href="https://www.youtube.com/watch?v=d04U7SjORTI" target="_blank">here</a>.

The most common response that I've gotten is: "That all sounds great, but I'll never be able to convince everyone in my company to start using a monorepo". And this is the reason that I'm writing this blogpost. Because moving to a monorepo doesn't have to be a big bang change at all. In fact, this is something you could do gradually, at your own pace over a long time period (although, I would try to do this in a reasonable time frame).

## Migrating

### Current situation

In most companies that I've visited over the past years, the situation is as follows. There are multiple teams working on different applications or libraries. Every team has its own repository where they store the code that is written by that team. If code needs to be shared with different teams, some code is published to an (internal) npm registry by one team and loaded by another team as a dependency.

This situation could be referred to as a being a 'polyrepo approach' and kind of looks like this:

![polyrepo](https://www.dropbox.com/s/lbf2k9rrd16mo42/polyrepo-migration-to-nx.png?raw=1)

In this scenario, we have three teams with three repos. Team 1 is working on a 'ui-kit' that is being published to npm. The other teams load this 'ui-kit' by downloading it from the (internal) npm registry. 

**Note:** Explaining why publishing to npm might not be the best idea in every situation and how a monorepo fixes this is not in the scope of this blogpost. For that, I would like to refer you to this <a href="https://www.youtube.com/watch?v=q4XmAy6_ucw" target="_blank">excellent talk</a> by Manfred Steyer.

### Hybrid situation

Most people think that, when you want to move to a monorepo, all three teams in the example above need to do that in a single go. But this isn't true at all. In fact, there is an obvious step between a polyrepo and a monorepo which I like to refer to as being a 'hybrid approach'. And it looks a little like this:

![hybrid](https://www.dropbox.com/s/uxpdfxim81oz2x4/hybrid-migrating-to-nx.png?raw=1)

In this specific scenario, as a first step towards a monorepo, the team responsible for 'app 1' and the team responsible for the 'ui-kit' have moved there code into a single repository. This repo is setup using Nx (hence the distinction between apps and libs inside of that repo). 

The important thing to notice here is the fact that the ui-kit is both published to npm AND imported inside of 'app 1'. This means that both inside the monorepo as outside of the monorepo this library is being used. This is exactly what is meant by the hybrid approach.

This allows us to gradually start with a 'monorepo'. You can start with two teams, and as soon as these get accustomed to it and everything is starting to run smoothly, a new team can be added. In the mean time, libraries that need to be shared with teams outside of the monorepo can still be published to npm. The end goal is that everything is moved to the monorepo.

It is also an ideal way to convince the non-believers of this approach. If people see that two teams can work perfectly well together inside of a single repo, you have a viable and working example to sell the idea!

### Complete monorepo situation

When everyone has move to the monorepo, the situation could look like this:

![monorepo	](https://www.dropbox.com/s/wyc5703yts5ddp2/monorepo-migrating-to-nx.png?raw=1)

Here, we no longer use npm as a means to distribute code between different teams. 

## Conclusion

Migrating to a Nx or a monorepo is something you can do one small step at a time. After having demonstrated everything works for a small amount of teams/projects/code, you can convince everyone that this is in fact they way to go (not always of course) and keep on migrating towards a complete monorepo!
