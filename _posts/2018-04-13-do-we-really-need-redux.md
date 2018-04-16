---
layout: post
title: Do we really need Redux or @ngrx/store
published: true
description: This article explains the cases when Redux is a good idea or when it is overkill for your application
author: brechtbilliet
comments: true
date: 2018-04-13
subclass: 'post'
categories: 'brechtbilliet'
disqus: true
tags: Redux
cover: 'assets/images/cover/cover1.jpg'
---

## About this article

If you are writing Angular, Vue or React applications, chances are big that you have used or encountered the Redux pattern.
Redux helps us to structure statemanagement in an immutable matter which is great, but in order to use it correctly, we have to write a bunch of boilerplate code.
In this article I would like to tackle the question on when to use Redux and why.
The referred plugins/code samples are written in an Angular context. But the principles explained in this articles work for other frameworks as well. This also means that we are using the [@ngrx/store](https://github.com/ngrx/platform/blob/master/docs/store/README.md) package instead of the [Redux](https://redux.js.org/) package

## To Redux or not to Redux?

First of all it's important to note that Redux solves certain problems for us. If we are not experiencing those problems, then there is a big chance that the Redux pattern
is overkill for our application.

The first question that we might want to ask ourselves is:
**Does My application have state?** State can be the value of a pager that we want to remember, or the fact that a sidebar is collapsed or not. State could be a cached set of data coming from our backend, or user information that we need throughout the whole application.
It could be a simple value that we want to remember in memory when we are navigating between pages.

Let's sum up some examples where the Redux principle might shine in our applications:
- Storing state (like the value of a search filter so it’s still available when the user navigates back to a certain grid)
- Sharing state between components that have their own route, and thus won’t have a parent component to pass them the state through inputs or properties
- Optimistic updates: Check [this article](https://blog.strongbrew.io/Cancellable-optimistic-updates-in-Angular2-and-Redux/)
- Real-time updates: Check [this article](https://blog.strongbrew.io/How-we-made-our-app-real-time-in-6-lines-of-code/)
- When we want undo/redo logic
- When we want to keep track of all the state changes and debug them with awesome tooling (Redux devtools)
- When we want an organised way of handling session storage or localstorage. Check [this plugin](https://github.com/btroncone/ngrx-store-localstorage)

When we are writing Angular applications it’s a best-practice to work with immutable data structures. That way we can make use of the [OnPush changedetectionstrategy](https://angular-2-training-book.rangle.io/handout/change-detection/change_detection_strategy_onpush.html) that angular provides us, which results in better performance and less unexpected behavior. In React we could use a [Pure component](https://reactjs.org/docs/react-api.html#reactpurecomponent) for that. So, for this article let's assume that immutable datastructures are the way to go and that our application needs it.

While we know that Redux forces you to use immutable datastructures, let's still challenge the need of Redux shall we?
If statemanagement is trivial to our applications and we just want to store values we could work with a state service instead of Redux.

```typescript
@Injectable()
export class UsersService {
    private _users$ = new BehaviorSubject([]);

    get users$(): Observable<User[]> {
        return this._users$.asObservable();
    }

    // IMPORTANT: since we use an immutable dataflow
    // we have to make sure users is a new instance
    setUsers(users: User[]): void {
        this._users$.next(...users);
    }
}

```

This example shows how we can set the simple value of an array of users by calling the `setUsers()`function. It will store the value into a BehaviorSubject which we will consume as an observable. The spread operator (`...`) will create a new instance of the `users` array. That way we know that we are working in an immutable manner. This was pretty easy and we don't need Redux anymore, nor all the boilerplate that we would had to write. So if the state of our application only contains a few simple properties, the Redux pattern might be overkill.

But what if we need to do more then just set the value of `users`. What if we want to add and remove users from that stream, and we would have to do it in an immutable manner? Checkout the following example for instance.

```typescript
@Injectable()
export class UsersService {
    private _users$ = new BehaviorSubject([]);

    get users$(): Observable<User[]> {
        return this._users$.asObservable();
    }

    setUsers(users: User[]): void {
        this._users$.next(...users);
    }

    addUser(user: User): void {
        // We cannot use array.push because we only want to
        // pass immutable data to the streame
        // for the OnPush strategy remmber?
        this._users$.next([...this._users$.getValue(), user])
    }

    removeUser(id: string) {
        // Again, we have to create a new Array instance to not break the
        // immutable dataflow
        this._users$.next(this._users$.getValue().filter(v => v.id !== id));
    }
}

```
The code above is starting to feel a bit weird, and it seems like we are writing reducer logic inside of this state service to keep it immutable. We have also created our own observable implementation so we could subscribe to the changes of our state service. While it could still be overkill to use redux if this is the only state in our application, it might become complex if we are working with multiple states, nested states, etc.

Another example is caching. People use redux to cache data results. A simple `shareReplay` operator might to the trick as well.

```typescript
fetchUsers(): Observable<User[]> {
    ...
    return this.httpClient.get('').pipe(shareReplay(1));
}
```

## Summary

I would suggest to not use Redux untill we actually need it, and in my experience most applications that I have written in the past did need redux at a certain point. That being said I also wrote a bunch of applications that didn't need it at all (CRUD applications for instance.

It's up to you if you want to use Redux or not, **but keep your applications immutable at all times**.
It will save you a lot of energy in debugging and it makes sure that you can optimise the change detection cycle in your applications.
