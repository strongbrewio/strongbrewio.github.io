---
layout: post
cover: false
title: combineReducers enhanced
date:   2016-09-15
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
cover: 'assets/images/cover/cover7.jpg'
tags: Redux
---
When working with redux or ngrx/store, you get a lovely utillity method called `combineReducers` that solves a pretty annoying problem for you. This method could however be further improved and we'll see why in a bit.

**Note: some knowledge of the redux architecture is required to continue with this blogpost. Checkout <a href="http://redux.js.org/" target="_blank">this</a> for more info on Redux.**

**Note: Since the PR still isn't merged in the @ngrx/store library, I created a separate npm module with the functionality described below. You can find the github repo <a href="https://github.com/KwintenP/combine-reducers-enhanced" target="_blank">here</a>.**

**TL;DR Go to the [conclusion](#conclusion)**

### Current situation
Take a look at the following state design.

```typescript
{
	"ui": {
		"topBarCollapsed": true,
		"sideBarCollapsed": true
	},
	"data": {
		"tweets": [],
		"users": []
	}
}
```

Let's try to translate this into reducers.

##### Without combineReducers

Remember, when your store is being initialised, you can only pass it a single reducer. To avoid a single reducer with all the logic, we are going to use the concept of reducer composition. Your reducer tree could look like this:

<img src="https://www.dropbox.com/s/gglg3j3ama5affo/Screenshot%202016-09-14%2016.35.12.png?raw=1" />

You can then create your store like this:

```typescript
// pseudo code
new Store(rootReducer);
```

The function of the rootReducer is:

- Delegate the actions towards the dataReducer and the uiReducer with the correct state slice.
- Assure that the reference of the data does not change if changes are made to the ui property and vice versa.

The 'rootReducer' doesn't do that much and it's a PITA to have to write this every time. Every time a new action is added, you need to add it to the rootReducer as well.
Enter `combineReducers`!

##### With combineReducers
`combineReducers` removes the need to write this reducer yourself. When creating your store, you can pass it, not only a reducer function, but also an object like this:

```typescript
const rootReducer = {
	ui: uiReducer,
	data: dataReducer
}

new Store(rootReducer);
```

The store will see this is not a function, and thus not a 'rootReducer'. Internally it will  pass this object to the `combineReducers` method. This will generate a 'meta-reducer' for you that does exactly the same as the 'rootReducer' we've described before.
Conceptually speaking it looks like this:

<img src="https://www.dropbox.com/s/i1mh7frvr8tddg0/Screenshot%202016-09-15%2007.42.26.png?raw=1" />

The advantage is that you do not need to write the 'rootReducer' yourself.
Pretty neat right!

### Problem description
Let's say, you have a state tree that looks more like this.

```typescript
{
	"ui": {
		"mainPage": {
				"topBarCollapsed": true
		},
		"loginPage": {
			"sideBarCollapsed": true
		}
	},
	"data": {
		"tweets": [],
		"users": []
	}
}
```

As you can see, we have a second nested level for our ui property and maybe we want to split the data into two different reducers, a tweets- and usersReducer.
Let's take a look at how we can implement this.

### Current possible solutions
The `combineReducers` method only allows a single level of nesting, so we'll have to see how we can fix this.

##### Create the intermediary reducers yourself
You can always decide to just implement the intermediate reducers yourself.

<img src="https://www.dropbox.com/s/e2gjsrcp03cxzq9/Screenshot%202016-09-15%2020.11.55.png?raw=1" />

However, writing these reducers yourself is the same PITA as it was for the first level so it's not the preferred solution

##### Use a third-party library which handles this for you
There is a third-party library created by Brecht Billiet that does exactly this. Check it out <a href="https://github.com/brechtbilliet/create-reducer-tree">here</a>.
You can pass it an object and it will create all the intermediate reducers for you. This is an awesome solution, but unfortunately, it requires us to depend an yet another third-party library.

##### Nest the combineReducers method yourself

Since we have the `combineReducers` method, we might as well leverage it fully and use it for every intermediary reducer. You can do this by passing an object like this to your store at creation time.

```typescript
const rootReducer = {
	"ui": combineReducers({
		"mainPage": mainPageReducer
		"loginPage": loginReducer
	}),
	"data": combineReducers({
		"tweets": tweetsReducer
		"users": usersReducer
	})
}
```

It will generate all the intermediary reducers for you but the 'rootReducer' definition doesn't look clean. For people just starting with Redux, this will probably look a little confusing as well.

It turns out, the `combineReducers` method can be changed to fix this pretty easily.
<a name="conclusion"></a>

### Proposed solution
I've created a <a href="https://github.com/ngrx/store/pull/214" target="_blank">PR</a> towards the ngrx/store library where I extended the `combineReducers` method to work with nested objects as well. This allows you to create an object like this:

```typescript
const rootReducer = {
	"ui": {
		"mainPage": mainPageReducer
		"loginPage": loginReducer
	},
	"data": {
		"tweets": tweetsReducer
		"users": usersReducer
	}
}

new Store(rootReducer);
```

which will create this:

<img src="https://www.dropbox.com/s/lpd3io77pqemecp/Screenshot%202016-09-15%2007.42.41.png?raw=1" />

It's actually a change of only two LOC and semantically does exactly the same as nesting the `combineReducers` method yourself or the third-party library, but it's a lot cleaner to implement and use.
If you feel this feature might help you in the future, be sure to 'like' the PR it so it could be added to the library asap.

**Note: As soon as the PR is being approved for the NGRX/Store library, I'll submit a similar one to Redux.js**















