---
title: RxJS best practices in Angular
published: true
author: brechtbilliet
description: This article is all about the do's and don'ts when it comes to writing reactive applications with RxJS in Angular applications. 
layout: post
navigation: True
date:   2018-01-04
tags: RxJS Angular
subclass: 'post'
categories: 'brechtbilliet'
logo: 'assets/images/strongbrewlogo.png'
disqus: true
cover: 'assets/images/cover/cover4.jpg'
---

This article is all about the do's and don'ts when it comes to writing reactive applications with [RxJS](http://reactivex.io/) in [Angular](https://angular.io/) applications. 
The best practices described in this article are based on personal experiences and can be assumed as personal opinions. 

The topics we will tackle in this article are:
- [Learning how to think reactive](#learning-how-to-think-reactive)
- [Pipeable operators](#pipeable-operators)
- [ASCII marble diagrams](#ascii-marble-diagrams)
- [Using pure functions](#using-pure-functions)
- [Avoiding memory leaks](#avoiding-memory-leaks)
- [Avoiding nested subscribes](#avoiding-nested-subscribes)
- [Avoiding manual subscribes in Angular](#avoiding-manual-subscribes-in-Angular)
- [Don't pass streams to components directly](#dont-pass-streams-to-components-directly)
- [Don't pass streams to services](#dont-pass-streams-to-services)
- [Sharing subscriptions](#sharing-subscriptions)
- [When to use Subjects](#when-to-use-Subjects)
- [Clean-code practices](#cleancode-practices)
- [Angular embraces RxJS](#Angular-embraces-RxJS)

**Note:**
We will refer to observables as streams in this article.
Since the streams in this article use the `$`-suffix, a short explanation.
First of all, there is a lot of debate about the `$`-suffix but I believe this should be a personal preference. 
The reason why I prefer to use it, is because I find it very easy to separate streams from regular objects.
That being said, I would not consider it a best practice, just a personal choice.

## Learning how to think reactive

Reactive programming is completely different than imperative programming. It requires us to make a certain mind switch.
This mind switch is rather important if we want to benefit from RxJS completely.
We want to **stop thinking in specific actions** and we want to **start thinking in streams**. 
It requires us to forget a part of practices that we already know (at least for a moment).
In [this article](http://blog.brecht.io/Creating-reactive-calendar-in-angular4/) we can find some tips and practical examples on how to start thinking reactive in RxJS.

## Pipeable operators

The first best practice is the use of pipeable operators. The operators being used in this article are pipeable.
Since version 5.5 RxJS has introduced these so called pipeable operators which are easier to import than patch operators, and
also have [treeshaking](https://webpack.js.org/guides/tree-shaking/) advantages. More information about pipeable operators can be found [here](https://blog.angularindepth.com/rxjs-understanding-lettable-operators-fe74dda186d3) and [here](https://blog.hackages.io/rxjs-5-5-piping-all-the-things-9d469d1b3f44).

This example illustrates the difference between doing it the old way and the new way.

```typescript 
// BAD: This is the old way and should be avoided (patch operators)
// as we can see the operators (filter, map) are part of the
// Observable prototype
import 'rxjs/add/operator/filter';
import 'rxjs/add/operator/map';
const new$ = Observable.interval$
    .filter(v => v % 2 === 0)
    .map(v => v * 2);

// GOOD: This is the new and improved way (lettable operators)
// we just use the pipe operator where we pass operators that
// we can import from 'rxjs/operators'
import {filter, map} from 'rxjs/operators';
const new$ = interval$
    .pipe(
        filter(v => v % 2 === 0),
        map(v => v *2)
    )
```

## ASCII marble diagrams

Some developers tend to say: "Great code should be self-explanatory, writing documentation is something that we might want to avoid."
In some cases I would agree with that statement, but for complex RxJS code we might want to reconsider.
Streams can become complex in the following scenarios:
- When we take the lifecycle of streams into account, (how long do they live? when do they start living? what destroys them?)
- When we start combining streams (every stream has a different lifecycle remember?)
- When we subscribe multiple times or even subscribe after a while, or even never subscribe to them

[marble diagrams](http://rxmarbles.com/) are a cool way of visualising streams but it's hard to put those marble-diagrams in our code right?!
There is an ASCII variant of these marble-diagrams that we can use to describe and document our complex streams and how they interact with each other.

ASCII diagrams have more advantages then just documenting:
- It gives us a graphic thinking model
- It becomes easy to review someones code and validate to see if it really does what it's supposed to be doing
- Great to draw on a whiteboard before we start coding
- You can type them in your IDE or editor before you actually start coding. (An easy way to trick your mind into thinking reactively)
- We can use them to write unit tests as well: [Checkout this awesome article](http://blog.kwintenp.com/how-to-setup-marble-testing/)

The concepts behind ASCII marble documentation are quite simple. Take this easy example for instance:

```typescript
// ---a--b--c--d---e---...
// ---a--b--c--d---e|
// ---a--b--c--d---e#
// ---a--b-^-c--d---e
```
- `-` (stands for a time frame)
- `a-z` (are the values that are next'ed in the stream)
- `|` (indicates that the stream has completed)
- `...` (indicates that the stream will keep on living)
- `#` (indicates that an error occurred)
- `^` (indicates where we start subscribing (only for hot streams)

Perhaps it's time to check a real example and how we might document it:

```typescript
const interval$ = interval(1000)            // 0--1--2--3--4--5--6...
const new$ = interval$
    .pipe(
        skip(1),                            // ---1--2--3--4--5--6...
        take(5),                            // ---1--2--3--4--5|
        filter(v => v % 2 === 0),           // ------2-----4---|
        map(v => v + 1)                     // ------3-----5---|
    )
```

Take a minute to let this sink into your brain, because this might be **THE WAY** of making a complex code snippets readable for anyone.
When we take a look at this diagram, it's fairly easy to comprehend what happens, and how every operator affects the `new$` stream we can see above. There is no "one way of doing things" when it comes to writing ASCII marble-diagrams. You can put them where and how you want.
As we want to do for all other documentation: **keep it up to date!**

## Using pure functions

RxJS follows the concepts of functional reactive programming which basically means that we will use [pure functions](https://medium.com/javascript-scene/master-the-javascript-interview-what-is-a-pure-function-d1c076bec976) to create our reactive flow.
A function is pure when:
- It doesn't mutate anything
- It will always return the same value based on the same parameters
- It doesn't have any side effects. It can't mutate state outside of the function

In the beginning it might seem pragmatic to use side effects, but that mostly means we aren't fully thinking reactively. 
Therefore avoid side effects at much as possible.

## Avoiding memory leaks

To consume a stream we need to **subscribe** to that stream. When we subscribe to that stream a **subscription** will be created.
That subscription will keep on living until the stream is **completed** or until we **unsubscribe manually** from that stream.
Managing subscriptions is very important and in a number of cases we will have to manually unsubscribe an existing subscription to avoid memory leaks. Take this example for instance:

```typescript 
class AppComponent implements OnInit {
   ngOnInit() {
        // The following stream will produce values every second
        // 0--1--2--3--4--5--6--...
        const interval$ = interval(1000);
        // Even when this component gets destroyed,
        // the stream will keep producing values...
        // This means the console will keep on logging
        // This is a classic example of a memory-leak
        const subscription = interval$.subscribe(r => console.log(r));
    }
}
```

To remove the memory-leak in this component we can keep track of the subscriptions by taking advantage of the `ngOnDestroy()` lifecycle hook of Angular:

```typescript 
class AppComponent implements OnInit, OnDestroy {
    subscriptions = [];
    ngOnInit() {
        const interval$ = interval(1000);
        const subscription = interval$.subscribe(r => console.log(r));
        // manually keep track of the subscriptions in a subscription array
        this.subscriptions.push(subscription);
    }

    ngOnDestroy() {
        // when the component get's destroyed, unsubscribe all the subscriptions
        this.subscriptions.forEach(sub => sub.unsubscribe());
    }
}
```

However, when we are using a bunch of subscriptions, it can become quite dirty. Before, we talked about the fact that a subscription will live until we manually unsubscribe (like we just did in the snippet above), but also until the stream gets **completed**. A cool way to handle this issue is to use a Subject that we next in the `ngOnDestroy()` lifecycle hook of Angular:

```typescript 
class AppComponent implements OnInit, OnDestroy {
    destroy$ = new Subject();
    ngOnInit() {
        // interval$: 0--1--2--3--4--5--6--...
        // destroy$:  -------------true|
        // result:    0--1--2--3--4|
        const interval$ = interval(1000);
        interval$
            // let the interval$ stream live 
            // until the destroy$ Subject gets a value
            .pipe(takeUntil(this.destroy$))
            .subscribe(r => console.log(r));
    }

    ngOnDestroy() {
        // when the component get's destroyed, pass something to the
        // destroy$ Subject
        this.destroy$.next(true);
    }
}
```

## Avoiding nested subscribes

Nesting subscribes is something that needs to be avoided as much as possible. It makes the code unreadable, complex, and introduces side effects.
It basically forces you to **NOT** think reactively. Take this Angular example for instance:

```typescript
class AppComponent {
    user: User;
    constructor(
        private route: ActivatedRoute, 
        private userService: UserService)
    {
        // when the params of the route changes,
        // we want to fetch the user and set the user property
        //
        // VERY BAD: nesting subscribes is ugly and takes away
        // the control over a stream
        this.route.params
            .pipe(map(v => v.id))
            .subscribe(id => 
                this.userService.fetchById(id)
                    .subscribe(user => this.user = user))
    }
}
```

The previous implementation is considered a bad-practice. It's recommended to use **higher-order streams** like `mergeMap` or `switchMap`. Let's have a look at this example:

```typescript
class AppComponent {
    user: User;
    constructor(
        private route: ActivatedRoute, 
        private userService: UserService) 
    {
        // when the params of the route changes,
        // we want to fetch the user and set the user property
        //
        // GOOD: we have created a single subscribe which makes
        // the flow way easier and gives us the control we need
        this.route.params
            .pipe(
                map(v => v.id),
                switchMap(id => this.userService.fetchById(id))
            )
            .subscribe(user => this.user = user)
    }
}
```

## Avoiding manual subscribes in Angular

To consume a stream we need to subscribe that stream, that's simply how observables work. But what if a component needs values from 5 different streams... Would that mean, that we want to subscribe to all of these streams and manually map all the values to unique properties, just to make it work? That would suck, right?!

Angular has this super cool feature called the `async pipe`. It's used to consume streams directly in the template
The async pipe does 3 things for us:
- It subscribes to the stream and passes the value to a component
- It **unsubscribes automatically** when the component gets destroyed (removes a lot of unsubscribe logic)
- Triggers change detection automatically

This means we don't have to manually subscribe nor unsubscribe anymore. Which cleans up the code a lot.
Let's have a look at the cleaned up previous example:

```typescript
@Component({
    ...
    template: `
        <user-detail [user]="user$|async"></user-detail>
    `
})
class AppComponent {
    // expose a user$ stream that will be 
    // subscribed in the template with the async pipe
    user$ = this.route.params.pipe(
        map(v => v.id),
        switchMap(id => this.userService.fetchById(id))
    );

    constructor(
        private route: ActivatedRoute, 
        private userService: UserService) {
    }
}
```

If you are into [React](https://reactjs.org/), I've created this lib called [react-rx-connect](https://www.npmjs.com/package/react-rx-connect) that would solve this problem. It binds the streams to the state, and unsubscribes from them when the component gets destroyed.

## Don't pass streams to components directly

One of the most important aspects of software architecture might be the concept of **decoupling** pieces of code.
Therefore we could consider passing streams to child components as a **bad practice** because it creates a very tight link between the parent component and the child component. They are no longer decoupled since subscriptions in the child component might trigger actions in the parent component. We never want the child component to be responsible of initiating data calls right?! That's the task of the smart component. See the difference between
[smart and dumb components here](http://blog.brecht.io/components-demystified/#smart-vs-dumb-components).
A component should always receive an object or value and should not even care if that object or value comes from a stream or not.

```typescript
// BAD
// app.component.ts
@Component({
    selector: 'app',
    template: `
        <!-- 
            BAD: The users$ steram is passed
            to user-detail directly as a stream 
        -->
        <user-detail [user$]="user$"></user-detail>
    `
})
class AppComponent {
    // this http call will get called when the 
    // user-detail component subscribes to users$
    // We don't want that
    users$ = this.http.get(...);
    ...
}

// user-detail.component.ts
@Component({
    selector: 'user-detail',
    template: `
        {{user|json}}
    `
})
class UserDetailComponent implements OnInit {
    @Input() user$: Observable<User>;
    user: User;
    ngOnInit(){
        // WHOOPS! This child component subscribes to the stream
        // of the parent component which will do an automatic XHR call
        // because Angular HTTP returns a cold stream
        this.user$.subscribe(u => this.user = u);
    }
}
```

It would be better to handle the subscription in the parent component itself:

```typescript
// GOOD
// app.component.ts
@Component({
    selector: 'app',
    template: `
        <user-detail [user]="user$|async"></user-detail>
    `
})
class AppComponent implements OnInit {
    users$: Observable<User[]> = this.http.get(...);
    user: User;
    ngOnInit(){
        // the app component (smart) subscribes to the user$ which will
        // do an XHR call here
        this.users$ = this.http.get(...);
    }
    ...
}

// user-detail.component.ts
@Component({
    selector: 'user-detail',
    template: `
        {{user|json}}
    `
})
class UserDetailComponent {
    // This component doesn't even know that we are using RxJS which
    // results in better decoupling
    @Input() user: User;
}
```

The responsibility of the component is clear. The user-detail is meant to be dumb and is completely decoupled from its parent.

There are however situations where we would like to create a stream from an input. In that case we could take a look at this library: [ngx-reactivetoolkit](https://www.npmjs.com/package/ngx-reactivetoolkit)

## Don't pass streams to services

Although, it might seem like a pragmatic solution to pass streams directly to services, it could be seen as a **bad practice** if we consider the decoupling again. By passing a stream to a service we don't know what's going to happen to it. The stream could be subscribed to, or even combined with another stream that has a longer lifecycle, that could eventually determine the state of our application.
Subscriptions might trigger unwanted behavior. And after all, services don't care that your components are using streams. Take this example for instance:

```typescript
// BAD
// app.component.ts
class AppComponent {
     users$ = this.http.get(...)
     filteredusers$ = this.fooService
        .filterUsers(this.users$); // Passing stream directly: BAD
    ...
}

// foo.service.ts
class FooService {
    // return a stream based on a stream
    // BAD! because we don't know what will happen here
    filterUsers(users$: Observable<User[]>): Observable<User[]> {
        return users$.pipe(
            map(users => users.filter(user => user.age >= 18))
    }
}
```

It would be better to use higher order streams for these situations.
Use `switchMap` over `mergeMap` if possible, since it will unsubscribe the previous stream.
The following example is better since all the RxJS logic is centralized in one place where the subscribing and unsubscribing happens: The smart component.

```typescript
// GOOD
// app.component.ts
class AppComponent {
    users$ = this.http.get(...)
    filteredusers$ = this.users$
        .pipe(switchMap(users => this.fooService.filterUsers(users)));
    ...
}

// foo.service.ts
class FooService {
    // this is way cleaner: this service doesn't even know
    // about streams now
    filterUsers(users: User[]): User[] {
        return users.filter(user => user.age >= 18);
    }
}
```

## Sharing subscriptions

Since most streams are cold by default, every subscription will trigger the **producer** of these streams.
The execution of the producer logic on every subscription, might not be what we want if we have multiple subscriptions.
Eg. Subscribing to Angular its `http.get()` multiple times will actually perform multiple xhr calls.
The following example will trigger the xhr call twice because `numberOfUsers$` depends on `users$`.

```typescript
@Component({
    selector: 'app',
    template: `
        Number of users: {{"{{numberOfUsers$|async"}}}}
        <users-grid [users]="users$|async"></users-grid>
    `
})
// BAD
class AppComponent {
    users$ = this.http.get(...)
    // the subscription on this stream will execute the xhr call again
    numberOfUsers$ = this.users$.pipe(map(v => v.length); 
}
```

In those cases we might want to share the subscriptions. The following example uses the `share()` operator:

```typescript
@Component({
    selector: 'app',
    template: `
        Number of users: {{"{{numberOfUsers$|async"}}}}
        <users-grid [users]="users$|async"></users-grid>
    `
})
// GOOD
class AppComponent {
    users$ = this.http.get(...).pipe(share());
    // the subscription on this stream will execute the xhr call again
    numberOfUsers$ = this.users$.pipe(map(v => v.length); 
}
```

Sharing a stream makes it hot. This means that if we subscribe after the value is produced, we will miss that value.
In that case we might want to use `shareReplay(1)` instead of `share()`. This will keep the last value in memory for us.

It's a common mistake to share everything. We don't always want to work with hot streams and sharing subscriptions comes with a small performance cost.Also, lazy streams have their advantages.

Angular also provides a *great alternative* that can reduce the sharing of streams to a minimum by using the `async as else` syntax.. 
Personally I would consider the use of this feature as a best practice.
The following example reduces the number of streams, the number of subscriptions and gives us **an easy way to show a loading indicator**.

```typescript
@Component({
    selector: 'app',
    template: `
        <div *ngIf="users$|async as users; else loading">
            Number of users: {{users.length}}
            <users-grid [users]="users"></users-grid>
        </div>
        <ng-template #loading>Loading...</ng-template>
    `
})
class AppComponent {
    // This stream will only subscribed to once
    users$ = this.http.get(...);
}
```

## When to use Subjects

A Subject is both a hot observable and an observer at the same time. This gives us the opportunity to next values into the stream ourselves.
Subjects tend to be overused by people that didn't make the mind switch towards reactive programming yet.

Only use them when really needed, for instance it's ok to use Subjects in the following scenarios:
#### When mocking streams in tests

```typescript
const fetchAll$ = new Subject(); // use a Subject as a mock
usersServiceMock.fetchAll.mockReturnValue(fetchAll$);
fetchAll$.next(fakeUser);
```

#### When we want to create streams from outputs in Angular

```typescript
@Component({
    ...
    template: `
    <some-component (search)="search$.next($event)"></some-component>
    `
})
class AppComponent {
search$ = new Subject(); // ----t-----te-----ter----term...
}
```

#### When handling circular references

I'm not going to dive in this to deep, but [Dominic Elm]() does an awesome job explaining this in [this great article](https://blog.thoughtram.io/rxjs/2017/08/24/taming-snakes-with-reactive-streams.html#behaviorsubject-to-the-rescue)


For most other cases an operator or Observable.create might be enough.

**Note:**
A BehaviorSubject is commonly used because it has a `getValue()` function. That would also be considered a bad practice.
When we are trying to fetch a specific value it usually means we are not thinking reactive.

## Clean code practices
Consistent code indentation and formatting can improve the readability of complex streams:
- Align operators below each other

```typescript
    foo$.pipe(
        map(...)
        filter(...)
        tap(...)
    )
```

- Extract into different streams when it becomes unreadable
- Put complexer functionality in private methods (make the reactive flow clear)
- Avoid the use of brackets for readability, that's personal preference.


## Angular embraces RxJS

We already saw a glimpse of why Angular is a framework that really embraces the use of RxJS.
Therefore it's recommended to use the functionality that Angular provides.
- The `ActivatedRoute` has exposes a params stream.
- The Http and HttpClient both return streams
- The `Form` and `FormControl` both have a `valueChanges` property that returns a stream
- The async pipe is an awesome feature that really helps us to use the streams in our templates
- Using the `ngOnInit()` lifecycle function to initialize streams can help us for mocking purposes

## Conclusion

Still here? Awesome! We learned a lot! If this article interests you, you might want to check out the "Advanced RxJS in Angular workshop" from [Strongbrew](https://strongbrew.io), where me and [Kwinten Pisman](blog.kwintenp.com) teach how to use advanced RxJS in real Angular applications.
