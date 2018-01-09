---
layout: post
cover: 'assets/images/posts/zone.png'
title: How the hell does zone.js really work?
date:   2016-06-09
subclass: 'post'
categories: 'kwintenp'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: Angular
cover: 'assets/images/cover/cover3.jpg'
---
If you've read about Angular 2 change detection, you will probably have heard about zones. Zones is a feature that was ported from Dart and is used internally by Angular 2 to determine if a change detection cycle should be triggered.

If you go to the <a href="https://github.com/angular/zone.js/" target="_blank">github</a> page of zone.js, you can find the following definition of a Zone:

> A Zone is an execution context that persists across async tasks. You can think of it as thread-local storage for JavaScript VMs.

If you're like me, this will probably not make too much sense the first time you read it. To better understand what this means, I recommend watching this talk by Brian Ford <a href="https://www.youtube.com/watch?v=3IqtmUscE_U" target="_blank">at ngConf 2014</a> and read this post by thoughtram on <a href="http://blog.thoughtram.io/angular/2016/01/22/understanding-zones.html" target="_blank">understanding zones</a>.
However, even after watching the talk and reading the blogpost, I was still intrigued to find out how this actually works. How can zone.js monkey-patch browser events and how do the examples provided on their github page really work. The purpose of this blogpost is to share what I've learned during my discovery.

##### How are events monkey-patched and what does that even mean?
To see how events are monkey-patched, I decided to look into the source code. The following is a simplified conceptual snippet of what zone.js does at startup time.

```typescript
function zoneAwareAddEventListener() {...}
function zoneAwareRemoveEventListener() {...}
function zoneAwarePromise() {...}
function patchTimeout() {...}
window.prototype.addEventListener = zoneAwareAddEventListener;
window.prototype.removeEventListener = zoneAwareRemoveEventListener;
window.prototype.promise = zoneAwarePromise;
window.prototype.setTimeout = patchTimeout;
```
**NOTE:** zone.js patches even more events which I omitted since the concept is the same.

It turns out, zone.js overrides some of the functions on the window prototype and replaces the defaults with a proxy. This means that every event scheduled or every promise created after loading the file, will be wrapped in the proxy. This concept is called monkey-patching.

##### Time to explain by example
Lets take a look at the first example from the readme of the zone.js github repository (Here's the <a href="http://plnkr.co/edit/Ul44DohpdBfMKjgssspw?p=preview" target="_blank">plnkr</a> you can play around with).

```typescript
//load zone.js
Zone.current.fork({}).run(function () {
    Zone.current.inTheZone = true;

    setTimeout(function () {
        console.log('in the zone: ' + !!Zone.current.inTheZone);
    }, 0);
});

console.log('in the zone: ' + !!Zone.current.inTheZone);
```

If you were to execute this, you'd get the following result:

```typescript
'in the zone: false'
'in the zone: true'
```

Normally, you'd expect the result of both statements to be true since we're logging the same property in both places.

To understand how this works, we need to zoom in on some parts of the snippet.

##### Creating and running something in a Zone
```typescript
Zone.current.fork({}).run( .... );
```
When zone.js is loaded, it creates a global property which you can use to access the root Zone. In this example, a Zone is created by forking the root Zone via the `Zone.current` property. We call the run function on the created object to run something **inside this Zone**.

<img src="http://www.jacquitalbot.com/wp-content/uploads/2013/11/in-the-zone.jpg" />

##### The function executed in the Zone
Next we look at the function that is run inside the Zone.

```typescript
....
Zone.current.inTheZone = true;

setTimeout(function () {
        console.log('in the zone: ' + !!Zone.current.inTheZone);
    }, 0);
....
```

It first attaches a boolean to the `Zone.current` property. It then sets up a timer to log the property after the call stack has been cleared (if you do not know what this means, I suggest looking at the following <a href="https://www.youtube.com/watch?v=8aGhZQkoFbQ" target="_blank">talk</a>).
##### The log statement outside of the Zone
Lastly, the same statement is logged **outside of the zone**.

```typescript
....
console.log('in the zone: ' + !!Zone.current.inTheZone);
```

We again point to the same `Zone.current` property. How could this not log the same result if we're pointing to the same property in both log statements?

##### Zone setup and teardown
Every time something is run **inside a Zone** or **a monkey-patched event is triggered**, the Zone or the proxy **sets up the Zone** before executing the function or callbacks. The proxy is able to setup the Zone since it holds **a reference to the Zone in which it was created**.
During setup, the state linked to this specific Zone is restored so that even timeouts, event listener, ... are executed like they were executed immediately. You could think of a Zone as being an **execution context that persists across async tasks**, as the definition states.

To further clarify, take a look at the next snippet. I re-arranged the code in the way it is executed and added zone setup and teardown points. Look at the comments for more information.

```typescript
// load zone.js. As stated before,
// this will monkey-patch all browser events.

Zone.current.fork({}).run(function () {
    // SETUP ZONE
    // trigger: A call to the run function is made. This will first
    //          setup the zone before executing the function param.
    // actions:
    //      - Zone.current property is set to the Zone
    //        in which the function is executed. In this case, it's
    //        the one we creating by forking the root Zone. Let's
    //        call it exampleZone from this point onwards.
    //      - Zone lifecycle hooks are called (we'll get back
    //        to that later).

    // a boolean is attached to the Zone.current property. Due
    // to the zone setup, this points to the exampleZone.
    Zone.current.inTheZone = true;

    // a timeout is registered. This isn't the 'default' timeout
    // since these are monkey-patched. So here, we are actually
    // setting up the proxy. What's important to remember is that
    // this proxy will hold a reference to the Zone in which it
    // was created, in this case the 'exampleZone'. This will be
    // used later.
    setTimeout(
       ...., 0);

    // TEARDOWN ZONE
    // trigger: The function that was to be executed
    //          inside the Zone has ended.
    // actions:
    //      - Zone.current property is reset to the root Zone
    //      - Zone lifecycle hooks are called
});

// Log statement. The Zone.current property is currently
// pointing to the root Zone. Since this does not know an
// 'inTheZone' property, this will log false.
console.log('in the zone: ' + !!Zone.current.inTheZone);

// The stack is cleared and the timer callback is executed

// SETUP ZONE
// trigger: The monkey-patched event is fired. The proxy wrapper
//          will trigger a Zone setup. Remember that the proxy
//          wrapper holds a reference to the Zone in which it
//          was created.
// actions:
//      - Zone.current property is set to the exampleZone (the
//        proxy holds a reference to the exampleZone)
//      - Zone lifecycle hooks are called
function () {
        // The exampleZone does contain the 'inTheZone'
        // property. So this will log true.
        console.log('in the zone: ' + !!Zone.current.inTheZone);
}
// TEARDOWN ZONE
    // trigger: The event callback has ended and the proxy
    //          does a Zone teardown.
    // actions:
    //      - Zone.current property is reset to the root Zone
    //      - Zone lifecycle hooks are called
```

Zone.js is able to setup and teardown the Zone when executing the timeout callback thanks to the monkey-patching of that event.
This should clear things up a bit!

###### How does Angular 2 leverage zones
I took a look inside the Angular 2 source code to determine how they leverage zones. Take a look at the next snippet:

```typescript
....
new NgZoneImpl({
      trace: enableLongStackTrace,
      onEnter: () => {
        // console.log('ZONE.enter', this._nesting, this._isStable);
        this._nesting++;
        if (this._isStable) {
          this._isStable = false;
          this._onUnstable.emit(null);
        }
      },
      onLeave: () => {
        this._nesting--;
        // console.log('ZONE.leave', this._nesting, this._isStable);
        this._checkStable();
      },
      setMicrotask: (hasMicrotasks: boolean) => {
        this._hasPendingMicrotasks = hasMicrotasks;
        this._checkStable();
      },
      setMacrotask: (hasMacrotasks: boolean) =>
            { this._hasPendingMacrotasks = hasMacrotasks; },
      onError: (error: NgZoneError) =>
            this._onErrorEvents.emit(error)
    });
....
```

This is from the <a href="https://github.com/angular/angular/blob/master/packages/core/src/zone/ng_zone.ts" target="_blank">NgZone.ts</a> source file. Zone.js exposes lifecycle hooks. This is a listing of the events Angular 2 listens to. Since everything in Angular 2 is run in a single Zone, ngZone, Angular 2 can leverage this to determine when it should perform a Change Detection cycle based on those callbacks. This removes the need to manually call `$digest` like in Angular 1.
Pretty neat right!
