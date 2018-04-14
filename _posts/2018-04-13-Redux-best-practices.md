layout: post
title: Redux (@ngrx/store) best practices
published: false
description: This article is all about what to do and what not to do when using Redux (@ngrx/store)
author: brechtbilliet
comments: true
date: 2016-12-21
subclass: 'post'
categories: 'brechtbilliet'
disqus: true
tags: Redux, @ngrx, Angular
cover: 'assets/images/cover/cover2.jpg'
---

[@ngrx/store](https://github.com/ngrx/platform/blob/master/docs/store/README.md) is a library that tries to solve the problems of statemanagement through the principles of [Redux](https://redux.js.org/). The difference between Redux and @ngrx/store is that @ngrx/store is written specifically for Angular and it embraces the use of Observables from [RxJS](http://reactivex.io/rxjs/).
The combination of redux principles and RxJS can be very powerful when it comes to writing reactive applications.
Since a lot of Angular projects use @ngrx/store, it might be a good idea to write down some best-practices.

Note: The best-practices and opinions described in this article are strictly personal. Best practices are almost always a matter of opinion. Nevertheless, we (StrongBrew) are using these best practices at all our customers on a daily basis and they certainly work for us.
From now on @ngrx/store will be reffered to as Redux in this article.

## To Redux or not to Redux?

The first question that we might want to ask ourselves is do we really need Redux in our application.
It is a best practice to only use it when your application demands it.
I have written [an article](https://blog.strongbrew.io/do-we-really-need-redux) that tackles this question separately.

## Basic best practices

While these are common sense for an experienced Redux developer, let's sum those up as a refreshment for the sake of completeness.
- Our application can only count one store, otherwise it would become to complex 
- Reducers have to be pure, this is a princpile from functional programming which makes functions predictable and avoids side effects
- Immutable datastructures are very important to optimise change detection cycles and avoid unexpected behavior, therefore reducers should handle data in an immutable manner

## Don't add models to the store

A model can be seen as a javascript object which has functionality, like the following example:
```typescript
export class User{
    constructor(private firstName: string, private lastName:string){
    }

    get fullName(): string {
        return `${this.firstName} ${this.lastName}`; 
    }
}
```

While the Redux package written by Dan Abramov forbids sending prototyped objects as a payload, @ngrx/store does not forbid it yet.
However, it is a bad practice because it adds a lot of complexity to the store and chances are big that the models will get broken because of the immutable way of handling data. Check this example for instance:

```typescript
const user = new User('Brecht', 'Billiet');
console.log(user.fullName); // Brecht Billiet
const updatedUser = {...user, lastName: 'Doe'};
console.log(updatedUser.fullName); // undefined
```

Since we have updated the user in an immutable way, it has created a new reference and thus all the functionality is lost.
This is exactly what our reducers will do with the data that flows into them.

## What do we put in the store?

We shouldn't put things in the store just because we can. We have to think about what state needs to be in the store and why.
State that is being shared between components can sometimes be kept in the parent component for instance. When state needs to be shared between different root components (rendered inside a router-outlet) we might want to keep that state into the store.

When we need to remember a value when navigating through the application we could put that in the store as well. An example here could be: Remembering if a sidebar was collapsed or not, so when we navigate back to the page with the sidebar, it would still be collapsed.

## Don't forget about router params

A common mistake is putting things inside the store that could easily be added in the url.
The benifit of keeping state in the url is:

- We can use the browser its navigation buttons
- We can bookmark the url
- We can share that url with other people

Example: We have a map of a city with houses. On every house that we click we have to zoom to that house and the information of that house could be shown in a sidebar
![map](/assets/images/posts/redux-best-practices/map.png)

## Designing the state

### Keep it flat

One of the most common bad practices is deep-nesting the state into something that becomes quite complex:

```typscript

// this is an example of how not to design state
export interface ApplicationState {
    moduleA: {
        data: {
            foo: {
                bar: {
                    users: User[],
                    cars: Car[]
                }
            }
        }
    }
}  

// keeping it flat makes the application way easier
export interface ApplicationState {
    users: User[],
    cars: Car[]
}  
```

I'm not saying you can not nest state, I am saying we have to be very carefull when we do. The general rule of thumb here is: "keep the state as flat as possible"
In @ngrx/store we can work with feature module reducers and lazy load them

### Using combineReducers in an AOT compatible way

Sometimes we do need some kind of nesting in our state. In that case we can use the `combineReducers` function that @ngrx/store provides us. It is important to use this function the correct way, so we can avoid AOT issues. The approach below shows how to nest reducers in an AOT compatible way

```typescript
import {combineReducers} from '@ngrx/store';
export function aReducer(state = 'a', action){
  return state;
}
export function bReducer(state = 'b', action){
  return state;
}
export function cReducer(state = 'c', action){
  return state;
}
export function dReducer(state = 'd', action){
  return state;
}


export const uiReducer = combineReducers({
  a: aReducer,
  b: bReducer
})


export const dataReducer = combineReducers({
  c: cReducer,
  d: dReducer
})


export const rootReducer = {
  ui: uiReducer,
  data: dataReducer
}
```

### Make it readonly

We already covered the reason why we need to work immutable, but how can we enforce this?
Typescript comes with a readonly keyword which we can use to make a property readonly

```typescript
export type User = {
    readonly firstName: string;
    readonly lastName: string;
}

const user: User = {firstName: 'Brecht', lastName: 'Billiet'};
user.lastName = 'Doe';//cannot assign to 'lastName' 
// because it is a constant of read-only property
```

## Action design
payload
classes
typesafety

## Reducer design
Create child reducers when it becomes too complex
No business logic

## Decoupling redux from the presentation layer

Having the store injected everywhere in our application is not a good idea. We want to create an Angular, Vue or React application. Not a Redux application.
Therefore we could consider the following as best practices:
- Components don't need to know we are using Redux, don't inject the store in them.
- Services generally don't need to know we are using Redux, don't inject the store in them.
- We want to be able to refactor Redux out of our application without to much effort

Therefore we want to have some kind of abstraction layer between the presentation layer and the state management layer.

## Redux as a messaging bus VS redux as a statemanagement layer

This might be a personal preference, but I like to use Redux as a pure statemanagement layer. Yes, there are tools like @ngrx/effects where
we can send actions to our application and those actions won't just perform statemanagement but will do XHR calls among other things.

The nice thing about this approach is that we use some kind of messaging bus. However, I mostly like to keep it simple and abstract Redux away as much as possible. Therefore I don't use @ngrx/effects and only use Redux to update pieces of state and consume theses pieces. Some part of me believes that Redux shouldn't be used to perform backend calls nor decide when to optimistically update. I usually tackle optimistic updates [this way].(https://blog.strongbrew.io/Cancellable-optimistic-updates-in-Angular2-and-Redux/).

That being said, I wouldn't call my approach a best practice, but it is a best practice to really think about which way we want it.