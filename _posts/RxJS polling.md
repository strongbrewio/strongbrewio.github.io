### RxJS polling

Polling is a common scenario in a lot of Single Page Applications. You want your user to see the latest data without them taking any actions. In some scenarios, you might even want to display this data realtime. In most cases however, this is overkill and requires changed at the backend of your application. Polling is a really good near immediate alternative.

Polling is something where RxJS really shines. We will look at different polling strategies and how we can implement them.

**Note:** The examples in this post will use Angular but the concepts can be ported everywhere.

### Simple polling

First we will take a look at a simple example where we want to fetch new data every 5 seconds. Lets first try and think about what we need. 

- a backend call
- a trigger that tells us when we need to execute our backend call

Executing a backend call is easy. We can create a stream that will execute a backend call when subscribed to like this.

```typescript
const lukeSkyWalker$ = this.http.get('https://swapi.co/api/people/1');
```

Next thing we need is a trigger that will tell us when it is time to fetch our data. In a world without RxJS we would probably use `setInterval`. This method allows us to pass it a callback that gets executed every 'x' seconds. 
With RxJS however, we have to change the way we think. We can no longer think in terms of callbacks, we have to think in terms of streams. If we apply this to the trigger we need, we want a stream that fires every 'x' seconds. 
Drawn in a ASCII marble diagram, this is what we want:

```
----1----2----3----4----5...
```

RxJS has a static `interval` method that will create this streams for us. We can pass it a number which will denote the time between the events.

```typescript
const trigger$ = interval(1000);
```

Now we have the two streams that we need, it is time to combine them. If you think about it, we basically want to re-execute our `lukeSkyWalker$` every time our `trigger$` fires. We want to map our trigger value to another observable/async action. To do that, we need to use a flattening operator. In our case, we want to execute the `lukeSkyWalker$` every time and not cancel it. In that case, we can use the `concatMap` operator. 

```
----1----2----3----4----5...
    \    \    \    \    \ 
     -l|  -l|  --l| --l| -l| 
------l----l-----l----l---l
```

```typescript
this.polledLukeSkyWalker$ = trigger$.concatMap(_ => this.lukeSkyWalker$);

```









