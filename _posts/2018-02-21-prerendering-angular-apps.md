---
layout: post
title: Prerendering angular applications
published: false
author: brechtbilliet
description: 
comments: true
---

## Foreword

There are several ways of optimizing Angular applications. We could compile them ahead of time through AOT-compilation.
We could use service-workers to optimize caching. And there are plenty of other PWA (progressive web-app) features as well that can increase the quality of our app.

However, there are a still a few problems that these optimizations won't fix:
- SEO Search engine optimization: The application is harder to index by search engines because the content isn't available on load time. Therefore the application is likely to fail on several SEO requirements.
- Initial page load: Since the application still needs to be bootstrapped after the pageload, there is a waiting time untill the user can use the application

These two problems can be fixed by SSR (Server side rendering). SSR takes the best of both worlds: We still have a SPA (single page application) that gives us the native looking javascript app but the page is rendered instantly. So it's rich and fast at the same time!

For our [StrongBrew](https://strongbrew.io) website we tried to use SSR. It was pretty great when we ran it locally, however the StrongBrew website is hosted on Firebase and the SSR part was hosted by Firebase functions. We really love Firebase and everything it stands for, but SSR on Firebase functions was really slow. Sometimes it took 4 seconds to serve the content...

Since the loading time of a website is very important if we want to keep our visitors, we had to find another way to serve the content in a more effective manner. The key to SSR is Angular Universal, and so it turns out we can use just that to prerender all the html at build-time instead of at run-time. The result went from several seconds to 30 miliseconds.

This is a super fast and super effective improvement but it has one important limitation.
**It's not possible for apps with dynamic content**. The data in the StrongBrew website isn't fetched with AJAX calls (at least not the data that has to be indexed), it rather works with simple webpack imports of JSON files. These are inserted at build time :)

## Let's dive in

Enough chit chat! Let's dive into some code!
I've created this [github repository](https://github.com/strongbrewio/prerender-angular-example) just for you! It's a simple website with a few pages that doesn't know how to prerender yet.
Checkout the branch `runtime` by running the command `git checkout runtime`. When running `npm i && npm run start` the application should install all its NPM dependencies and get hosted on http://localhost:4000, just like any default Angular-CLI application

## Angular Universal
todo