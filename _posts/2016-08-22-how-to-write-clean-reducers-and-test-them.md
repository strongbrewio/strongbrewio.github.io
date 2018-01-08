---
layout: post
cover: false
title: How to write clean reducers (and test them!)
date:   2016-08-22
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: Redux
---
Last week, someone asked me how I kept my reducers clean and how to properly test them. Since it wasn't the first time someone asked me that, I decided to write it down for future reference.
Here it is!

**NOTE: Reducers are a part of the Redux architecture. If you do not know what I'm talking about, checkout <a href="http://redux.js.org/" target="_blank">this</a>.**

### Reducers basic building blocks
 To me, a clean reducer has a number of different building blocks. Let's take a look at the following example. It's a very simple tweetsReducer that manages an array of tweets.

```typescript
function tweetsReducer(
        state: State = [],
        action: Action): State {
    let tweet: Tweet;
    switch (action.type) {
        case "ADD_TWEET":
            ({tweet} = action.payload);
            return [...state, tweet];
        default:
            return state;
    }
}
```

**NOTE: all examples are written using typescript**

#### Default values

First thing we're going to focus on, is the default value for the input parameter.

```typescript
state: State = [];
```

To understand why defining a default parameter is important, you need to know about the lifecycle of Redux. Every Redux implementation has a method called `createStore` or something similar. This method creates the store.

<img src="https://cdn.meme.am/instances/500x/62970260.jpg" />

What this method also does is dispatch an action to that newly created store. If you've used Redux before, then you know that the Store holds a single State object. At startup, this object doesn't exist yet and in order to create it, every Redux implementation will dispatch an `INIT`-like action to get all the default values from the reducers and build the state tree.

Next is a simplified, conceptual snippet that shows what a `createStore` method could look like.

```typescript
function createStore(reducer: Reducer): Store {
    // Create the new store and pass the reducer
	const store: Store = new Store(reducer)

	// dispatch an init action to the store
	store.dispatch({type: "REDUX_INIT"});

	return store;
}
```

The dispatching of the `INIT`-action will result in the reducer, for example the tweetsReducer, being called with the following parameters.

```typescript
tweetsReducer(undefined, initActionDispatchedToTheStore: Action);
```

The first parameter should be the state, but since the State object is not yet created, it will be undefined. This results in the use of the default value of the state parameter, which in our tweetsReducer is an empty array. The initial value won't always be that easy, this can also be a complex object.

What's important to remember is to define default states to assure we get a nice, clean state object at startup.

**NOTE: Be sure to never define an action with the same name of your Redux implementation's `INIT`-action. You can easily check this in the source code.**

**NOTE: Often, you can pass an initial state object to the `createStore` method as a second parameter. This doesn't mean that you shouldn't add the default values to ALL your reducers**

#### Default case
The second part I want to focus on is the default case of the switch statement. This simply returns the state object it was given.

```typescript
default:
     return state;
```

In most cases, you will have multiple reducers in your application. These will be organised in a tree, matching your state object. Actions dispatched to the store will be delegated to the top reducers which will cascade, if necessary, the actions to the bottom reducers. The result of all these calls will form the new state object.
If you forget to add the default case to your reducers, part of your state tree will dissapear. Take a look at the following example and imagine we've forgotten to implement the default case in our tweetsReducer.

```typescript
// Current state tree where we manage an array of users
// and an array of tweets
let currentState: State = {
	"tweets": [
                 {
                    id: 1,
                    username:"@KwintenP",
                    content:"blogpost on clean reducers!"
                 }
	],
	"users": []
}

// we dispatch an action to add a user
store.dispatch({type: "ADD_USER", payload: {id: 1, name: "Kwinten"});
```

The store will send the action, not only to the usersReducers (left out for brevity), but also to our tweetsReducer.
When the action enters our tweetsReducer, none of the cases defined would match and since we have no default case, the reducer would return undefined. This would result in the following state object.

```typescript
{ "tweets": undefined, "users": [{id: 1, name: "Kwinten"}]};
```

As you can see, the tweets that were previously there, are gone. So, remember to always implement the default case to avoid that parts of your state tree disappear.

#### ES6 syntax sugar
A little bit of ES6 syntax sugar I like to use in my reducers is the following:

```typescript
//example payload
action.payload = {tweet: new Tweet(....)};

// we map the single value to the tweet property
let tweet: Tweet;
({tweet} = action.payload);
// Non ES6 equivalent
tweet = action.payload.tweet;

// example payload
action.payload = {tweet: new Tweet(...), id: 1};

// We map multiple values to the properties
let tweet: Tweet;
let id: string;
({tweet, id} = action.payload);
// non ES6 equivalent
tweet = action.payload.tweet;
id = action.payload.id;
```

I always use this ES6 shorthand in my reducers. In my tweetsReducer (see snippet at the beginning), I define a few variables above my switch statement. I then use these inside my case statements to map the payload to. The reason I do this is to make the reducer more readable. If I now wish to add a tweet to my store, I just look at the top of my case statement and I can immediately see what parameters I need to add to the payload.

```typescript
case "ADD_TWEET":
        ({tweet} = action.payload); // my payload needs a tweet object
        return [...state, action.payload.tweet];
```

In this particular example this might not seem that handy, but once your reducers become (a lot) bigger, you'll be happy if you've done this.

**NOTE: If you're using action creators (out of scope for this post), and you should, the problem that it's not clear what parameters you should pass, becomes less applicable. But, I would strongly recommend to keep doing this for when you're writing the action creators themselves and for your unit tests.**

#### Use reducer composition
As soon as your application grows, your reducers will become more complex. To avoid having a single reducer that manages the entire state tree, you could use the concept of reducer composition.
Take a look at the next reducer (I removed the implementations of the case statements for brevity):

```typescript
export function tweetsReducer(state:Array<Tweet> = [], action:Action):Array<Tweet> {
    let id:number, tweet:Tweet, tweets: Array<Tweet>;
    switch (action.type) {
        // first block
        case ADD_TWEET: // implementation
        case REMOVE_TWEET: // implementation
        case SET_TWEETS: // implementation
        case UPDATE_TWEET: // implementation

        // second block
        case TWEET_UN_LIKED: // implementation
        case TWEET_LIKED: // implementation
        case TWEET_UN_RETWEETED: // implementation
        case TWEET_RETWEETED: // implementation
        case TOGGLE_STAR_TWEET: // implementation
        default:
            return state;
    }
}
```

If we were to implement every case in the same reducer, this would already become quite big. If we look closely, we can see that the cases can be divided into two categories. The first block, handle the tweets collection. The second block handles an individual tweet.
###### Enter reducer composition!
This is an ideal example to demonstrate when you can use reducer composition. We can make the tweetsReducer handle the collection, and make a new tweetReducer, which handles a single tweet. This would look like this:

```typescript
export function tweetsReducer(state:Array<Tweet> = [], action:Action):Array<Tweet> {
    let id:number, tweet:Tweet, tweets: Array<Tweet>;
    switch (action.type) {
        // collection block
        case ADD_TWEET: // implementation
        case REMOVE_TWEET: // implementation
        case SET_TWEETS: // implementation
        case UPDATE_TWEET: // implementation

        // single tweet block
        case TWEET_UN_LIKED:
        case TWEET_LIKED:
        case TWEET_UN_RETWEETED:
        case TWEET_RETWEETED:
        case TOGGLE_STAR_TWEET:
        // For the cases above, we delegate to the
        // tweetReducer. Here we loop the current
        // collection of tweets and if it's the one
        // we wish to update, we replace it with the
        // result of the tweetReducer.
            ({id} = action.payload);
            return state.map(tweet => tweet.id == id?
                   tweetReducer(tweet, {type: action.type, payload: {tweet}}):
                   tweet);
        default:
            return state;
    }
}

export function tweetReducer(state: Tweet = {}, action: Action) {
    let tweet: Tweet;
    switch (action.type) {
        case TOGGLE_STAR_TWEET: // implementation
        case TWEET_RETWEETED: // implementation
        case TWEET_UN_RETWEETED: // implementation
        case TWEET_LIKED: // implementation
        case TWEET_UN_LIKED: // implementation
        default: return state;
}

```

Working with reducer composition makes your individual reducers less complex to manage.

#### Other tips

1. You MUST keep your reducers immutable! You can use the new <a href="https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/Object/assign" target="_blank">'Object.assign'</a> and the <a href="https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Operators/Spread_operator" target="_blank">spread operator</a> from ES6 or libraries as <a href="https://facebook.github.io/immutable-js/" target="_blank">Immutable.js</a> or <a href="https://github.com/rtfeldman/seamless-immutable" target="_blank">seamless-immutable</a> to do this.
2. When you're using reducer composition, combine your delegating cases at the bottom of your reducer. In the example of reducer composition, all the calls to the tweetReducer are combined at the bottom. This is merely a convention that I've found useful in the past.

#### How to test them
Testing reducers is quite easy and straightforward. You send a state object and an action to the reducer en check your result. But, you cannot forget to test the immutability of your reducers. To ensure this, you can use a library called deepFreeze. Check out this next example.

```typescript
describe("reducer: tweetsReducer", () => {
    describe("on ADD_TWEET", () => {
        it("should add a tweet to the
           current list of tweets", () => {
            // create the initial state object
            let tweet: Tweet = new Tweet(...);
            let initialState: Array<Tweet> = [tweet];

            // perform a deepFreeze on the initial state object
            deepFreeze(initialState);

            // create the payload
            let tweetToAdd: Tweet = new Tweet(...);
            let payload: any = {tweet: tweetToAdd};

            // Sent the action to the tweetsReducer
            let changedState: Array<Tweet> = tweetsReducer(initialState,
                {
                    type: ADD_TWEET,
                    payload
                });

            // Verify the changes are correct
            expect(changedState.length).toBe(2);
            expect(changedState[1]).toBe(payload.tweet);
        });
    });
});
```

As you can see, testing reducers consists of four steps:
1. Create an `initialState` object.
2. Perform a deepFreeze on the `initialState` object. This library will perform a recursive `Object.freeze()` on the entire state object protecting it from mutation. This is an easily overlooked but extremely important step in the testing of your reducers!
3. Call the reducer with the initial state and the correct action.
4. Verify the result of your reducer.

At least one test should be added for every case in your reducer's switch statement. Besides those, you should also add a test for an unknown action to verify you've implemented the default case. And one more test for the `INIT`-action to make sure you've defined a default value for your state. In this particular test you pass undefined as your state value.

#### Summary
- Use **default values** for your state object in your reducer
- Implement the **default case** to just return the state
- Use ES6 syntax sugar to make your reducers look more clean
- Use **reducer composition** when necessary
- Reducers MUST be **immutable**
- Use **deepFreeze** while testing to **ensure immutability**
- Combine your delegating cases at the bottom of your reducer
