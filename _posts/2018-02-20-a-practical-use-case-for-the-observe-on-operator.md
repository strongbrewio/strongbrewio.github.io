---
title: A practical use case for the observe-on operator
published: true
author: kwintenp
description: This article shows a practical use case for the observe on operator
layout: post
navigation: True
date:   2018-02-20
tags: Angular
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
disqus: true
cover: 'assets/images/cover/cover4.jpg'
---
A few weeks ago, I needed to implement a use case where the user could enable a certain feature when he triple clicked on the screen. Leveraging the power of RxJS, I thought it would only take a couple of minutes to create. Turns out it was a little more difficult than expected. I even had to use the 'observeOn' operator to get everything working. I discovered a pattern where the operator would become usefull in other scenarios as well. Let me show you how we can get there in several steps.

// List all the operators they will learn

// Add a TL:DR;

### Creating a click stream

Creating a stream of clicks is fairly easy. We can use the 'fromEvent' operator for that. Let's look at the code:

```typescript
const clicks$ = Rx.Observable.fromEvent(document, 'click')
  .mapTo('click');
```

The 'fromEvent' operator takes the element you want to add an event listener to and the event name. Using the 'mapTo' operator, we transform the event into a string, which is a little more convenient when we log out the event.

### Listen for clicks close to each other

Multi clicks only happen when the clicks happen close together. We could for example say that we want to 'catch' all the clicks that happen until the user stops clicking for 250ms. To accomplish this, we can use two operators, 'buffer' and 'debounceTime'. 

'Buffer' allows us to catch all events until a certain event occurs. This way, we can 'capture' all clicks until the user stops clicking.
'DebounceTime' allows us to know when the user stopped clicking.










 