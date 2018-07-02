---
title: Redux (@ngrx/store) best practices
published: true
author: brechtbilliet
description: This article is all about what to do and what not to do when using Redux (@ngrx/store)
layout: post
navigation: True
date:   2018-04-27
tags: Redux @ngrx Angular
subclass: 'post'
categories: 'brechtbilliet'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover2.jpg'
---

[@ngrx/store](https://github.com/ngrx/platform/blob/master/docs/store/README.md) is a library that tries to solve the problems of state management through the principles of [Redux](https://redux.js.org/). The difference between Redux and @ngrx/store is that @ngrx/store is written specifically for [Angular](https://angular.io) and it embraces the use of Observables from [RxJS](http://reactivex.io/rxjs/).
The combination of redux principles and RxJS can be very powerful when it comes to writing reactive applications.
Since a lot of Angular projects use @ngrx/store, it might be a good idea to write down some best-practices.

Note: The best-practices and opinions described in this article are strictly personal. Best practices are almost always a matter of opinion. Nevertheless, we (StrongBrew) are using these best practices at all our customers on a daily basis and they certainly work for us.   From now on @ngrx/store will be reffered to as Redux in this article.

## To Redux or not to Redux?

The first question that we might want to ask ourselves is do we really need Redux in our application.
It is a best practice to only use it when your application demands it.
[This article](https://blog.strongbrew.io/do-we-really-need-redux) tackles this question separately.

## Basic best practices

While the following list might be common sense for an experienced Redux developer, let's sum those up as a refreshment for the sake of completeness.
- Our application can only count one store, otherwise it would become too complex 
- Reducers have to be pure, this is a principle from functional programming which makes functions predictable and avoids side effects
- Immutable datastructures are very important to optimise change detection cycles and avoid unexpected behavior, therefore reducers should handle data in an immutable manner
- Reducers always have to return a value! So don't forget to implement the default case of the switch statement to return the original state

## Don't add models to the store

A model can be seen as a javascript object which has functionality, like the following example:

```typescript
class User{
    constructor(private firstName: string, private lastName:string){
    }

    get fullName(): string {
        return `${this.firstName} ${this.lastName}`; 
    }
}
```

While the Redux package written by Dan Abramov forbids sending these prototyped objects as a payload, @ngrx/store does not forbid it yet.
However, it is a bad practice because it adds a lot of complexity to the store and chances are big that the models will get broken because of the immutable way of handling data. Check this example for instance:

```typescript
const user = new User('Brecht', 'Billiet');
console.log(user.fullName); // Brecht Billiet
const updatedUser = {...user, lastName: 'Doe'};
console.log(updatedUser.fullName); // undefined
```

Since we have updated the user in an immutable way, it has created a new reference and therefore all its functionality has been lost.
This is exactly what our reducers will do with the data that flows into them. So always send plain objects when it comes to sending payloads in the actions.

Another approach for models is using Interfaces. Interfaces are great because are only interpreted in compile time, doesn't use memory in runtime and are very expressive. If your model doesn't need to be computed (like `get fullName()` in the previous example), then is advisable to use Interfaces instead.

Using Interfaces allows us to get advantage of some TypeScript features like `Pick<T>`. With `Pick<T>` we can create Types from Interfaces, which is very handy in some situations, like creating a model and a record with fewer properties:

```typescript
export interface User {
    id: number;
    name: string;
    middleName: string;
    lastName: string;
    fullName: string;
    address: string;
    city: string;
    state: string;
    zip: string;
}

export type UserRecord = Pick<User, 'id' | 'fullName'>;
```

If the endpoint to get a list of users is returning a partial object instead of the full object then we can create a new Type instead of two Classes or Interfaces. This is a cleaner approach and easier to maintain if your models are prepared for your UI.

## What do we put in the store?

We shouldn't put things in the store just because we can. We have to think about what state needs to be in there and why.
State that is being shared between components can sometimes be kept in the parent component for instance. We call that inner state:
The component keeps its own state, the component itself is responsible for that. If that component state does not affect anything from the application state, it does not need to be on the application state or touch redux. 

However, when state needs to be shared between different root components (rendered inside a router-outlet) we might want to keep that state in the store.

When we need to remember a value when navigating through the application we could put that in the store as well. An example here could be: Remembering if a sidebar was collapsed or not, so when we navigate back to the page with the sidebar, it would still be collapsed.

Complex state is something that we might want to put in the store as well, since Redux can handle complex state management in an elegant way.
The general rule of thumb here could be, **Only keep shared state, values that we want to remember and complex state in the store**. Don't add state in the store if we don't need to, it would result in unneeded boilerplate and complexity.

That being said, there are 2 more reasons where we might want to add extra state into the store:
- When we want to make our application real-time. Check out [How we made our app real time in 6 lines of code](https://blog.strongbrew.io/How-we-made-our-app-real-time-in-6-lines-of-code/).
- When we want to do optimistic updates. Check out [Cancellable optimistic updates in Angular2 and Redux](https://blog.strongbrew.io/Cancellable-optimistic-updates-in-Angular2-and-Redux/)


## Don't forget about router params

A common mistake is putting things inside the store that could easily be added in the url.
The benefit of keeping state in the url is:

- We can use the browser navigation buttons
- We can bookmark the url
- We can share that url with other people

If we can put simple things into the url, we should at least consider it.

## Avoid HUGE lists

Redux can not be seen as a local in-memory database, so we can't put all our data into the store for performance reasons.
Redux can be seen as an abstraction of state and data that our application needs at a certain time.

For instance if we have a list of 10000 users, we don't want to put them all in the store. What we could do is keep track of a list of 500 users in the store, which the user can see at that specific time, and load more users on the background and update that buffered list.

## Designing the state

Designing the state of our application is an important step, and we recommend to draw that state on a whiteboard first. The most important rule here is: **Keep the state as flat as possible**

One of the most common bad practices is deep-nesting the state into something that becomes rather complex:

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

I'm not saying you cannot nest state, I am saying we have to be very careful when we do. The general rule of thumb here is: **keep the state as flat as possible**
If we want to compose state in @ngrx/store we can work with feature module reducers and lazy load them as we can see in [Feature Module State Composition](https://github.com/ngrx/platform/blob/master/docs/store/api.md#feature-module-state-composition).

## Make everything readonly

We already covered the reason why we need to work immutable, but how can we enforce this?
Typescript comes with a readonly keyword which we can use to make a property readonly

```typescript
type User = {
    readonly firstName: string;
    readonly lastName: string;
}

const user: User = {firstName: 'Brecht', lastName: 'Billiet'};
user.lastName = 'Doe';//cannot assign to 'lastName' 
// because it is a constant of read-only property
```

This would certainly make sure we aren't updating properties in our reducers by accident. It does suck that we have to write readonly for every property.
The cool thing is that typescript offers us something called "advanced types" where we can do something like this:

```typescript
// By using the Readonly<> advanced types all the properties inside the type
// are readonly by default
type User = Readonly<{
    firstName: string;
    lastName: string;
}>;
```

## Action design

### Actiontypes

An action type should be a string that explains what the action should change in the store. Keep these strings consistent. Don't make the actiontypes too long, keep them short and clear.

```typescript
// This is bad
const DATA_USERS_SET_USER_ADDRESS = 'DATA_USERS_SET_USER_ADDRESS';

// This is better
const SET_USER_ADDRESS = 'SET_USER_ADDRESS';

```

Another cool idea might be to suffix the action with square brackets and put the whole thing into an action object:

```typescript

// Easy to read/debug
const UserActions = {
  SET_ADDRESS: '[USER] ADDRESS'
}

```

If the state managment would become very large we could prefix the action, but let's keep it simple and small as long as we can.

### Action creator classes

When we use plain action types and payloads it becomes quite painful to remember all the action type names and all the payloads that belong to them. This example for instance:

```typescript
const user_id = '1234', address = {whatevz};
this.store.dispatch(
    {
        type: 'SET_USER_ADDRESS', 
        payload: {user_id, address}
    });
```

That's pretty nasty if we want remember all that stuff, so let's create action creator classes for these. What if we could do this?

```typescript
const user_id = '1234', address = {whatevz};
this.store.dispatch(new SetUserAddressAction(user_id, address));
```

That's just became way easier to use and we don't have to remember the payload of the action.

If we wanted to implement the actioncreator class for this action it would look like this:

```typescript
class SetUserAddressAction implements Action {
    type = SET_USER_ADDRESS;
    payload: {user_id: string, address: Address};
    constructor(user_id: string, address: Address){
        this.payload = {user_id, address};
    }
}
```

### Payload design

When the action would only have one property for the payload we might be encouraged to use the payload directly instead of creating a property in it. However that would lead to inconsistency, so it might be better to always use subproperties

```typescript
// This is bad (inconsistent with the rest of the actions)
class UpdateUserAction implements Action {
    type = UPDATE_USER;
    payload: User;
    constructor(user: User){
        this.payload = user;
    }
}
// This is better
class UpdateUserAction implements Action{
    type = UPDATE_USER;
    payload: {user: User};
    constructor(user: User){
        this.payload = {user};
    }
}
```

### Type Safety

Type Safety is a huge win when using Redux with typescript, it requires a bit of boilerplate but it makes developing reducers feel like a walk in the park. It makes sure that our applications won't compile if they have type errors and it gives us great autocompletion inside our reducers.
Therefore I would definitely consider it a must. Since [Kwinten Pisman](https://twitter.com/KwintenP) already wrote an [awesome article](https://blog.strongbrew.io/type-safe-actions-in-reducers/) about this we won't go in to much detail here.

## Reducer design

### Destructuring the payload

If we want to make the reducer code more readable and shorter we could use javascript destructuring for that.
This might be personal preference, but it sure as hell makes our reducers easier to read. Take this example for instance:

```typescript
function usersReducer
  (state: User[], action: UserActions): User []{
    switch(action.type) {
      case 'SET_USER_ADDRESS':
        return state.map(v => 
          v.id === action.payload.user_id ? 
          {...user, address: action.payload.address} : 
          v
        )
    }
}
```
The `action.payload.`code comes back a few times, resulting in longer codelines.
The following piece of code might be more readable:

```typescript
function usersReducer
  (state: User[], action: UserActions): User []{
    switch(action.type) {
      case 'SET_USER_ADDRESS': {
        const {user_id, address} = action.payload;
        return state.map(v => 
          v.id === user_id ? 
          {...user, address} : 
          v
        )
      }
    }
}
```

As we can see have have used destructuring to extract the properties of the payload into variables.
Cleaner right? Let's imagine that our actions has 5 or even more properties on their payloads. In that case this would definitely help.
Something to note here is that the case implementation is wrapped inside a block statement. This is important because our reducer can have the same payload properties for different actions.

This means that `user_id` and `address` won't be available in the other case statements, which is exactly what we want.

### Don't write business logic inside our reducers

Reducers should not contain business logic, they are used to handle the state in an immutable fashion. We won't write business logic inside reducers because:

- It would become very complex
- Business logic has nothing to do with state management
- We have services for that

### Child reducers

When reducers need to update a piece of state a few levels down in the tree it can become complex in no-time. Take this example for instance:

```typescript
type User = {
    id: string;
    contracts: Contract[];
}
type Contract = {
    id: string;
    assignees: Assignee[];
}
type ApplicationState = {
    users: User[];
}   
...
```

If we would put all the logic to add an assignee to a specific contract of a specific user, the code would be hard to read. Checkout the following piece of code:

```typescript
// This is bad
function usersReducer
  (state: User[], action: UserActions): User []{
  switch(action.type) {
    case 'ADD_USER_CONTRACT_ASSIGNEE': {
      const {user_id, contract_id, assignee} = action.payload;
      return state.map(v => 
        v.id === user_id ? 
        {
          ...user, 
          contracts: user.contracts.map(contract => 
            contract.id === contract_id ?
            {
              ...contract, 
              assignees: [...contract.assignees, assignee]
            } : 
            contract
          )
        } : 
        v
      )
    }
    default:
      return state;
  }
}
```

When reducers become complex it might be a good idea to split the reducer up into child reducers. Check the refactored version of the previous example:

```typescript
// This is better
function usersReducer
  (state: User[], action: UserActions): User []{
  switch(action.type) {
    case 'ADD_USER_CONTRACT_ASSIGNEE': {
      const {user_id, contract_id, assignee} = action.payload;
      return state.map(v => 
        v.id === user_id ? 
        {
          ...user, 
          contracts: contractsReducer(contracts, action.payload)
        } : 
        v
      )
    }
    default:
      return state;
  }
}

function contractsReducer
  (state: Contract[], action: UserActions): Contract []{
  switch(action.type) {
    case 'ADD_USER_CONTRACT_ASSIGNEE': {
      const { contract_id, assignee} = action.payload;
      return state.map(v => 
        v.id === contract_id ? 
        {
          ...contract, 
          assignees: [...assignees, assignee]
        } : 
        v
      )
    }
    default:
      return state;
  }
}

```

As we can see, we have extracted the handling of contracts into its own reducer, which follows the exact same principles of a regular reducer.

The example just became a lot easier to read and way more maintainable. When traversing complex data structures, reducer nesting can be a really elegant way of managing state.

## Testing

Since reducers are pure functions, unit testing them is very easy.
We won't need to mock out any dependencies and we only have to test the value that the reducer returns.

We can also use [deepfreeze](https://www.npmjs.com/package/deep-freeze) to freeze the state that when the reducer accidently mutates data, the tests will throw an error. Deepfreeze is nothing more than a recursive `Object.freeze`

```typescript
describe('reducer: usersReducer', () => {
  describe('case UPDATE_USER', () => {
    it('should return a new instance with the correct state', 
    () => {
      const initialState = [new User('1'), new User('2')];
      // deepfreeze makes sure the reducer 
      // doesn't mutate anything by accident
      deepfreeze(initialState); 
      const user = new User('2');
      const action = new UpdateUserAction(user);
      const newState = usersReducer(initialState, action);
      // check if the result of the array is a new ref
      expect(newState).not.toBe(initialState); 
      // check if the result of the user is a new ref
      expect(newState[1]).not.toBe(initialState[1]);
      // check if the user got updated automatically
      expect(newState[1]).toEqual(user);
    });
  });
});
  
```

**Note: Don't forget to test the default action**

## Decoupling redux from the presentation layer

Having the store injected everywhere in our application is not a good idea. We want to create an Angular, Vue or React application. Not a Redux application.

Therefore we could consider the following as best practices:
- Components don't need to know we are using Redux, don't inject the store in them.
- Services generally don't need to know we are using Redux, don't inject the store in them.
- We want to be able to refactor Redux away from our application without to much effort

Therefore we want to have some kind of abstraction layer between the presentation layer and the state management layer.

How to abstract away the statemanagement layer can be read in the following two articles: [A scalable angular architecture](https://blog.strongbrew.io/A-scalable-angular2-architecture/) and [A scalable angular architecture part 2](https://blog.strongbrew.io/A-scalable-angular-architecture-part2/).
This is an architecture that we are using at our customers that really works for us.

## Redux as a messaging bus VS redux as a state management layer

This might be a personal preference, but I like to use Redux as a pure state management layer. Yes, there are tools like @ngrx/effects where
we can send actions to our application and those actions won't just perform state management but will do XHR calls among other things.

The nice thing about this approach is that we use some kind of messaging bus. However, I mostly like to keep it simple and abstract Redux away as much as possible. Therefore I don't use @ngrx/effects and only use Redux to update pieces of state and consume theses pieces. Some part of me believes that Redux shouldn't be used to perform backend calls nor decide when to optimistically update. I usually tackle optimistic updates [this way](https://blog.strongbrew.io/Cancellable-optimistic-updates-in-Angular2-and-Redux/).

That being said, I wouldn't call my approach a best practice, but it is a best practice to really think about which way we want it.

## Conclusion

We learned a lot! Once again, the best practices explained in this article are based on personal experiences and projects we have worked on. These are practices that work for us. They are not meant to be seen as the only way of doing things.

## Special thanks

A very warm and much appreciated special thanks to the following people:

[Nicholas Jamieson](https://twitter.com/ncjamieson), [Raúl Jiménez](https://twitter.com/elecash) and [Fabian Gosebrink](https://twitter.com/FabianGosebrink): Thank you all so much for reviewing and pointing out some awesome ideas


Your input makes blogging worth while!
