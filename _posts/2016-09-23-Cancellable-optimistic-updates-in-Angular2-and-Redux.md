---
layout: post
title: Cancellable optimistic updates in Angular 2 and Redux
published: true
description: Angular 4 and redux] are 2 technologies that are getting a lot of traction these days. Angular 2 is a SPA (single-page-app) framework and redux is a state management tool. Most developers that are using Angular 2 are using the @ngrx/store variant of the redux principle. Because I'm one of them I will be using @ngrx/store instead of redux.js for this article. Don't let that bother you, what you are about to read can be used with redux.js in the exact same way.
author: brechtbilliet
comments: true
date: 2016-09-23
subclass: 'post'
categories: 'casper'
disqus: true
tags: Angular Redux
---

[Angular 2](https://angular.io/) and [redux](https://github.com/reactjs/redux) are 2 technologies that are getting a lot of traction these days. Angular 2 is a SPA (single-page-app) framework and redux is a state management tool. Most developers that are using Angular 2 are using the [@ngrx/store](https://github.com/ngrx/store) variant of the redux principle. Because I'm one of them I will be using @ngrx/store instead of redux.js for this article. Don't let that bother you, what you are about to read can be used with redux.js in the exact same way. If you have never heard about redux, I strongly advice to read the [documentation](http://redux.js.org/) first.

## Optimistic updates

Redux has a client-side store that has all the data and state your client-side application needs. Therefore, it is the single source of truth for your frontend. When something in that store changes, your components get updated automatically. Let's say that we are implementing a winecellar application which will obviously contain an array of wines.

**Scenario: we want to remove a wine from the winecellar.**

A user goes to the index page of our application, clicks a delete button in the datagrid which will call an angular service to remove a wine. The service does an HTTP call to a server, and when the wine finally gets deleted, the service responds with a 200 response (ok). When that happens we can update the redux store and our view gets updated.
![Scenario 1](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/optimisticupdates/optimisticupdates_scenario1.png)

The code explained above might look like this:

```typescript
remove(wine: Wine): void {
    this.http.delete(`${API_URL}/wines/${wine._id}`).subscribe(() => {
    	// dispatch the action to the store, when the call was successful
    	this.store.dispatch({type: REMOVE_WINE, payload: {_id: wine._id}});
    });
}
```

This is the traditional way of doing things, but I think we can do it better. What if we would update the store, regardless of the HTTP response the server returns? We click the delete button in the datagrid, call the angular service like we already did. But in that service we update the store directly (parallel with the http call)

![Scenario 2](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/optimisticupdates/optimisticupdates_scenario2.png)

The code explained above might look like this:

```typescript
remove(wine: Wine): void {
	// dispatch directly to the store
	this.store.dispatch({type: REMOVE_WINE, payload: {_id: wine._id}};
	// we still have to subscribe because http calls return cold observables
    this.http.delete(`${API_URL}/wines/${wine._id}`).subscribe();
}
```

This is a pretty nice change: Our application has just gotten a lot **faster and snappier**. Every time the user does an action, the store gets updated immediately, which means that the components will get updated immediately as well. We never have to wait for http responses again. 

There is one exception to this rule: When **adding** data, the backend has to return an ID which means that for POST calls we will have to postpone the store update until we get a 200 response. (otherwise we would have id-less wines in our store, and we can't have that)

## What if the server-call fails?

What if the user's internet-connection is lost? Or the backend doesn't return a 200 response but an error. Our store will already be updated, the wine will be lost when it's in fact still in the database. In some scenarios you want to prevent that kind of behavior. In those cases we want to rollback that specific action, but not interfere with the rest of the actions. It has to be completely safe.

![Scenario 3](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/optimisticupdates/optimisticupdates_scenario3.png)

We would like to implement something like this:

```typescript
remove(wine: Wine): void {
	// create an action
	let action = {type: REMOVE_WINE, payload: {_id: wine._id}};
	//dispatch the action to the store
	this.store.dispatch(action);
	// call the backend
    this.http.delete(`${API_URL}/wines/${wine._id}`)
    	.subscribe(
    		// on success, do nothing
    		() => {}, 
	    	// on error, rollback the action
	    	() => {
	    		this.store.dispatch({type: "UNDO_ACTION", payload: action});
	    	// maybe show somekind of errormessage to show the user that it's action failed
    	});
}
```

It turns out that when using the redux pattern, this only takes 12 lines of code (without comments :-)). We will have to create a parent reducer that will delegate to our root reducer. The parent reducer will keep track of all the actions, so they can be rolled back.
The implementation looks like this:

```typescript
import {ApplicationState} from "../statemanagement/state/ApplicationState";
import {Action, ActionReducer} from "@ngrx/store";

export function handleUndo(rootReducer: ActionReducer<ApplicationState>)
	: ActionReducer<ApplicationState> {
	// keep the executedActions
    let executedActions: Array<Action> = [];
    return (state: ApplicationState, action: Action) => {
        if (action.type === "UNDO_ACTION") {
        	// if the action is UNDO_ACTION, 
        	// then call all the actions again on the rootReducer, 
        	// except the one we want to rollback
            let newState: any = {};
            executedActions = executedActions.filter(eAct => eAct !== action.payload);
            // update the state for every action untill we get the
            // exact same state as before, but without the action we want to rollback
            executedActions.forEach(executedAction => 
            	newState = rootReducer(newState, executedAction));
            return newState;
        }
        // push every action that isn't an UNDO_ACTION to the executedActions property
        executedActions.push(action);
        // just delegate
        return rootReducer(state, action);
    };
}
```

So basically, when an action is being rolled back, every action that has been taken before is being executed on a piece of state again. When that piece of state is updated, it will return it and the store will be updated with the same state, except the one that our rolled back action created.

To make sure the actions can be undone, we have to make redux use the piece of code above. This is how you could use it in Angular 2.

```typescript
@NgModule({
	// instead of passing the rootReducer directly
	// like we would have done before, wrap it in the handleUndo function
    imports: [StoreModule.provideStore(handleUndo(rootReducer))/*, ...*/],
    /* ... */
})
export class AppModule {
}
```

## Possible improvements

Won't it become slow after a while, replaying all these actions?
Some facts:
<ul>
<li>Actions are very cheap</li>
<li>They will update the store only once, we execute the actions ourselves, so ui won't get updated</li>
<li>Actions only get replayed when there is an error</li>
<li>The devtools also work like that, enables [timetraveling](https://onsen.io/blog/react-redux-devtools-with-time-travel)</li>
</ul>
What if it would become slow?
We can implement a buffer. Let's say that we only want the last 100 actions to be kept.

```typescript
export function handleUndo(rootReducer: ActionReducer<ApplicationState>, 
	bufferSize = 100): ActionReducer<ApplicationState> {
    let executedActions: Array<Action> = [];
    let initialState = undefined;
    return (state: ApplicationState, action: Action) => {
        if (action.type === "UNDO_ACTION") {
        	// if the action is UNDO_ACTION, 
        	// then call all the actions again on the rootReducer, 
        	// except the one we want to rollback
            let newState: any = initialState;
            executedActions = executedActions.filter(eAct => eAct !== action.payload);
            // update the state for every action untill we get the
            // exact same state as before, but without the action we want to rollback
            executedActions.forEach(executedAction => 
            	newState = rootReducer(newState, executedAction));
            return newState;
        }
       	// push every action that isn't an UNDO_ACTION to the executedActions property
		executedActions.push(action);
        let updatedState =  rootReducer(state, action);
        if (executedActions.length === bufferSize + 1) {
            let firstAction = executedActions[0];
            // calculate the state x (buffersize) actions ago
            initialState = rootReducer(initialState, firstAction);
            // keep the correct actions
            executedActions = executedActions.slice(1, bufferSize + 1);
        }
        return updatedState;
    };
}
```

### Conclusion
The redux pattern has opened amazing doors for frontend development. Just like [realtime](http://blog.brecht.io/How-we-made-our-app-real-time-in-6-lines-of-code/) became a breeze we can do optimistic updates with almost no effort.
The reason why you would do optimistic updates is that your application becomes amazingly fast and snappy! When implementing optimistic updates, the user will experience a native, mobile feeling.

Here you can find the npm package I've created for angular, called [ngrx-undo](https://www.npmjs.com/package/ngrx-undo)

Let me know if you enjoyed this article! 
