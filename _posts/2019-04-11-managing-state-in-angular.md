---
layout: post
title: Managing state in Angular
published: true
description: 
author: brechtbilliet
comments: true
date: 2019-07-29
subclass: 'post'
categories: 'brechtbilliet'
disqus: true
tags: architecture
cover: 'assets/images/cover/cover4.jpg'
---

## About this article

When we create single-page-applications, there is no way around it. Sooner or later we are going to be facing state. There are a lot of solutions out there to manage that state and together with those solutions there are a lot of opinions.
Especially when working with experienced developers, there opinions get stronger and often result in interesting discussions.

This article is not about which libraries to use, but to learn about what state is and how we can reason about it. The examples are all written in Angular, however the approaches are not specific to Angular.

In this article we will learn about the different kind of state types within our application, and where that state might live.

## What is state?

State is basically everything that will define the UI that our user will be using.
State could be whether a button should be visible or not, it could be the result of that button click and it could also be an `Array` of users that is coming from an API.
State can live in different places throughout our entire application. Some state is very specific to a certain component where other state might be shared in different parts of our application. One piece of state could be a singleton instance, where a another piece of state could share the limited lifespan of a component that can be destroyed at any time.

This big variety of what state could be, how long it lives and where it comes from results in complexity that we need to manage.

## What is state management?

State management is the concept of adding, updating, removing and reading pieces of state in an application. When we have deeply nested data structures and we want to update a specific part deep down in the tree, it might become complex. In that case we have state management libraries that contain a `Store` which helps us with state management to get rid of that complexity. A quick note, we have to be careful that these libraries don't add complexity by overusing them.

## Reactive state

Combining state management together with reactive programming can be a really nice way to develop single-page-applications. Whether our focus lies on Angular, Vue or React, combining these two principles  will result in more predictable applications.

Now what has state to do with reactive programming?
A piece of state can change over time, so in a way we are waiting for new state changes. That makes it asynchronous.

Let's take this example for instance:

```typescript
// false------true-----false---true...
sidebarCollapsed$ = this.state.sidebarCollapsed$
```

The `sidebarCollapsed$` stream starts out with `false`, later on it becomes `true` and so on. This stream keeps on living. In Angular this state can be consumed with the [async pipe](https://angular.io/api/common/AsyncPipe) as easy as:

```html
<my-awesome-sidebar *ngIf="sidebarCollapsed$|async">
</my-awesome-sidebar>
```
The `async` pipe will subscribe to the `sidebarCollapsed$` pass it to the component, mark it for check and will automatically unsubscribe when the component gets destroyed.
Keeping state in an observer pattern is nice because we can subscribe to the changes. Oh, and did I mention it plays super nice with Angular?

We can either use a `BehaviorSubject` or state management frameworks that support Observables. Here are some really great ones with Observable support:

- [Ngrx](https://ngrx.io)
- [Akita](https://netbasal.gitbook.io/akita/)
- [Ngxs](https://github.com/ngxs/store)

## Immutability and Unidirectional data flow

Before we dive deeper in state, there are 2 important principles that we should follow when managing state.
The first principle is **immutability**, which means that we should never mutate data directly without creating a new reference of that object.
If we mutate data directly, our application becomes unpredictable and it's really hard to trace bugs.
When we work in an immutable fashion we can also take advantage of performance strategies like the [ChangeDetection.OnPush](https://netbasal.com/a-comprehensive-guide-to-angular-onpush-change-detection-strategy-5bac493074a4) from Angular or React its [PureComponent](https://reactjs.org/docs/react-api.html#reactpurecomponent).

When we use typescript we can enforce the typescript compiler to complain when we mutate data

```typescript
type Foo = {
    readonly bar: string; 
    readonly baz: number; 
}
let first = {bar: 'test', baz: 1};
first.bar = 'test2'; // compilation error
first = {...first, bar: 'test2'}; // success
```

In the previous example we have overwritten the `first` instance with an entire new instance that has an updated `bar` property.

Arrays can be handled like this:

```typescript
let arr = ['Brecht', 'Kwinten'];
arr.push('John'); // BAD: arr is mutated
arr = [...arr, 'John']; // Good, arr gets new reference
```
the `Array` prototype also has some great helper functions that we can use to enforce immutability like `map()` and `filter()` but this is not in scope for this article.

The second principle is **Unidirectional data flow**.
In a nutshell, this means that we should never use two-way data binding on state. It is the absolute owner of that specific piece of state that is in charge of updating it (immutable of course). 

Both of these principles are highly enforced by the [Redux](https://redux.js.org/) pattern.

## What kind of states are there?

### Router state

Often forgotten, but one of the most important pieces of state a web application can have. Putting state in the route gives us the following advantages:

- We can use the browser navigation buttons
- We can bookmark the state
- We can can copy and paste the url with the state to other users
- We don't have to manage it, it's always there in the route

**Tip**: Instead of handling modals with a `userDetailModalVisible` property, why not enjoy all the benefits mentioned above and bind it to a `users/:userId` route?
Using a child `router-outlet` in Angular makes this a piece of cake as we can see in this snippet.

```html
<table>
<!--contains users -->
</table>
<router-outlet>
<!-- user detail modal rendered in here -->
</router-outlet>
```

### Component state

Every component could contain state. That state could be shared with its dumb components or could be used in the component itself.
Eg: When an `ItemComponent` has a property `selectedItems` which is an array of ids, and that array is never used in other components (that aren't children of that component), we can consider it component state. 
It belongs to that component, therefore the component should be responsible for it. Child components can consume that state but should *never mutate it*. Those components can notify their parent that is responsible for it, which could update it in an immutable way. For more information about smart and dumb components [look here](https://blog.strongbrew.io/components-demystified/#smart-vs-dumb-components).

Personally, I try to avoid state management frameworks for managing component state because it's the responsibility of that component to manage that state.
There are however good reasons to use state management frameworks to manage component state:

- When the state management is very complex
- If we want to do [optimistic updates](https://blog.strongbrew.io/Cancellable-optimistic-updates-in-Angular2-and-Redux/) 
- If we want to use it for [realtime stuff](https://blog.strongbrew.io/How-we-made-our-app-real-time-in-6-lines-of-code/)

If the state management of the component becomes a bit too complex and we don't want to use a state management framework just yet, we could use a state reducer in the component itself.


### Persisted state

Persisted state, is state that is being remembered when the user navigates between different pages. This could be whether a sidebar was collapsed or not, or when the user returns to a grid with a lot of filters and he wants them to be remembered and reapplied when he returns.
Another example is a wizard with different steps, and every step needs to be persisted so the user can navigate back and forth and the last page is a result of all these steps.

Persisted state is the type of state where we typically use a state management framework for, that being said, if we don't want to rely on an external dependency we can also manage it in a Angular `service` which can be a singleton that is shared throughout the entire application. If that `service` becomes too complex or there is a lot of state to manage, I would consider to put that state into a state management framework.

### Shared state

When we are talking about shared state, we are talking about state that needs to be shared between different parts of our application. State that is being shared throughout different smart components. This means that the instance of this piece of state should live on a higher level, than the components that want to consume it.

Shared state can be managed in a state management framework like [Redux](https://redux.js.org/), [Ngrx](https://ngrx.io), [Akita](https://netbasal.gitbook.io/akita/), [Ngxs](https://github.com/ngxs/store) and so on, but if that state is small and simple we can also manage it manually.
Let's say that we want an `Observable` of an `Array` of countries that we need to share throughout the entire application. In Angular we could have a `CountryService` that fetches the countries from the API once, and then shares it throughout the entire application. 
For that we can use the `shareReplay` operator from RxJS.

```typescript
export class CountryService {
    ...
    countries$ = this.httpClient.get('countries').pipe(shareReplay(1));
}
```

Simple right, one line of code?! For this we don't **need** a state management framework, although it can also have its benefits.
Some developers like to keep all their master data in a `Redux` store, and that's fine. Just know that we don't have to.
I like to develop by the **KISS** principle (**K**eep **I**t **S**imple **S**tupid) as much as possible, so I favor this approach many times.
Think about the amount of lines of code we saved by this approach.
Beware that every line of code we write, not only needs to be written but also maintained.

## Which state needs to be managed?

Now that we know what state is, we have to ask ourselves which state needs to be managed, and where do we manage that state? In a component, singleton service or a framework (Store)?

This is the part where the strong opinions surface.
I would suggest to use what works for you and your team and really think about, but here are  **my personal opinionated** guidelines:

- I try to avoid state management frameworks where possible. RxJS already leverages us with a lot already and I like to think **KISS**.
- I try to avoid using state management frameworks to communicate with different parts in my application, I believe state is unrelated to communication.
- When my component can handle the state and it's not too complex, I let my component in charge of managing that state.
- Master data like countries are exposed in a service which uses the `shareReplay` operator.
- I don't put the result of a `getById` API call into a store if there is no one consuming that state except for the component requesting it
- I use a facade between my smart components and my store/services to make refactoring easier in the future.

However, there is also a popular opinion out there to put literally everything in the store which has the following advantages:

- We can see the flow of the code in devtools
- Consistent pattern
- We can leverage selectors with memoization
- Easier for realtime applications
- Optimistic updates are easier

However, there are a few downsides as well:

- A gigantic amount of bloat code: Bigger bundle size, more maintenance and dev time. Eg: If we would use the complete Ngrx pattern for the `countries$` example we would have to write an: `action`, `actiontype`, `effect` and a  `reducer`.
- Tightly coupled to a strong dependency that is hard to get rid of in the future
- Generally more complex
- The user his screen can get out of sync with the backend
- Cache invalidation: if we add a `currentUserToEdit` in the store, we have to get it out when we navigate away
- We can't use the `async` pipe to cancel pending XHR requests
- We create a distributed monolith of some sort

## Wrapping up

State management is a topic surrounded with discussions and opinions. There is no right or wrong, use what works for you and your team. There are awesome libraries out there, use them if it benefits you, but at least think about it before you use them. That's the goal of this article after all, to get everyone to start thinking about state managment rather than jumping to the first solution.

## Special thanks

Special thanks to the awesome reviewers:

[Tim Deschryver](https://twitter.com/tim_deschryver)
[Jeffrey Bosch](https://twitter.com/jefiozie)
[Ruben Vermeulen](https://twitter.com/rubverm)
