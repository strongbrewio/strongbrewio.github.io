---
layout: post
cover: false
navigation: True
title: The $onChanges lifecycle-hook in depth
date:   2016-04-06
tags: AngularJS
subclass: 'post'
categories: 'casper'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
author: kwintenp
---
With the release of Angular 1.5.3 we got some awesome new features. As you all know, the point of the Angular 1.5 releases are to bring us closer to the Angular 2 way of working. With that in mind, the latest new feature we've got are Lifecycle hooks. What those are and how you can use them has already been perfectly explained by Pascal Precht on [his blog](http://blog.thoughtram.io/angularjs/2016/03/29/exploring-angular-1.5-lifecycle-hooks.html). For the remainder of this blog, I'm going to assume you have read it.
I'm not going to try and do the same, instead I want to look at a specific lifecycle hook in depth, `$onChanges`. We'll look at how it is implemented by Angular, how we can use it with some examples, a pitfall when using it with non-primitives and how we can fix it.


#### How is it implemented in the Angular codebase
The `$onChanges` lifecycle hook can be used with the `'<'` and `'@'` bindings. So to get started, we must look at how Angular handles these bindings.

```javascript
case '<':
     if (!hasOwnProperty.call(attrs, attrName)) {
        if (optional) break;
            attrs[attrName] = void 0;
     }
     if (optional && !attrs[attrName]) break;
     parentGet = $parse(attrs[attrName]);
     destination[scopeName] = parentGet(scope);
     removeWatch = scope.$watch(
          parentGet,
          function parentValueWatchAction(newParentValue) {
                   var oldValue = destination[scopeName];
                   recordChanges(scopeName, newParentValue, oldValue);
                   destination[scopeName] = newParentValue;
          },
          parentGet.literal);
     removeWatchCollection.push(removeWatch);
     break;

```
**Note: We'll only look at the `'<'`binding since the concepts are the same.**

This is the entire code block that is executed when a one-way binding is found. The part that really interests us here, is the `scope.$watch` expression and how it is setup.

```javascript
scope.$watch(
     parentGet,
     function parentValueWatchAction(newParentValue) {
              var oldValue = destination[scopeName];
              recordChanges(scopeName, newParentValue, oldValue);
              destination[scopeName] = newParentValue;
            },
     parentGet.literal
);
```

Let's look at the listener function in particular, which is named `parentValueWatchAction` here. The listener function is the one that gets executed when the `$watch` expression is found dirty (if you do not understand what this means, I suggest you take a look at the [$digest cycle](https://www.ng-book.com/p/The-Digest-Loop-and-apply/) which is a very important concept in Angular 1).
We can see that, if a `$watch` expression is dirty and the listener function is fired, a method called `recordChanges(scopeName, newParentValue, oldValue);` is executed. Hmmm thats seems interesting, lets dig deeper.

```javascript
function recordChanges(key, currentValue, previousValue) {
        if (isFunction(destination.$onChanges)
             &&
            currentValue !== previousValue) {
          // If we have not already scheduled the top
          // level onChangesQueue handler then do so now
          if (!onChangesQueue) {
            scope.$$postDigest(flushOnChangesQueue);
            onChangesQueue = [];
          }
          // If we have not already queued a trigger
          // of onChanges for this controller then do so now
          if (!changes) {
            changes = {};
            onChangesQueue.push(triggerOnChangesHook);
          }
          // If the has been a change on this property
          // already then we need to reuse the previous value
          if (changes[key]) {
            previousValue = changes[key].previousValue;
          }
          // Store this change
          changes[key] = {
               previousValue: previousValue,
               currentValue: currentValue
          };
        }
}

function triggerOnChangesHook() {
        destination.$onChanges(changes);
        // Now clear the changes so that we schedule
        // onChanges when more changes arrive
        changes = undefined;
}
```

This function, which is executed every time a dirty watch expression is found, does a number of things.
What it basically does is: Check if your controller, called `destination` here, has an `$onChanges` method defined. If this is the case, it creates an object called `changes` with the previous value and the current value and pushes the `$onChanges` method with the `changes` object onto a queue.
It also registers a function called `flushOnChangesQueue` to be executed during the `$$postDigest` phase. Let's look at what this function does.

```javascript
// This function is called in a $$postDigest to trigger
// all the onChanges hooks in a single digest
function flushOnChangesQueue() {
      try {
        if (!(--onChangesTtl)) {
          // We have hit the TTL limit so reset everything
          onChangesQueue = undefined;
          throw $compileMinErr(
                'infchng',
                '{0} $onChanges() iterations reached.Aborting!\n',
                TTL);
        }
        // We must run this hook in an apply since
        // the $$postDigest runs outside apply
        $rootScope.$apply(function() {
          for (var i = 0, ii = onChangesQueue.length; i < ii; ++i) {
            onChangesQueue[i]();
          }
          // Reset the queue to trigger a new schedule
          // next time there is a change
          onChangesQueue = undefined;
        });
      } finally {
        onChangesTtl++;
      }
}
```
This function just loops over all the callbacks that were pushed on the queue earlier, and executes them. So every controller with the `$onChanges` callback defined, that has a bound value that was changed, is notified.

That's basically it! What's important to remember is:

* for the lifecycle hook to work, a `scope.$watch` expression must be dirty

* the lifecycle hooks that have values changed, are triggered after every digest cycle

* an object containing the old and the new value can be passed to the `$onChanges` callback


#### Time for some examples!

###### Primitives

We're going to start of with an easy example. I've create a [plnkr](http://plnkr.co/edit/ZB6CLEXuRMrEnTIjUpo7?p=preview) that contains two components. A  component called 'outer' that contains another component called 'inner'. Let's look at the 'outer' component first.

```typescript
class OuterComponent{
  public template: string =`
    <div>
      Value: {{"{{$ctrl.primitiveValue"}}}} <br />
      <input ng-model="$ctrl.primitiveValue"> <br /> <br />

      <inner primitive-value="$ctrl.primitiveValue"></inner>
    </div>
  `;
}
```

**Note: all the examples are written using Typescript.**

It's about as easy as it gets. If we look at it's template, we can see it has a property called `primitiveValue` that you can change via an input box. It also defines our 'inner' component and binds the `primitiveValue`.

Let's look at the 'inner' component.

```typescript
class InnerComponent {
  public controller: Function = InnerController;
  public bindings: any = {
    primitiveValue : "<"
  };
  public template: string =`
    <div>
      PrimitiveValue: {{"{{$ctrl.primitiveValue"}}}} <br />
      Number of times the $onChanges is invoked: {{"{{$ctrl.onChangesCalledCounter"}}}}
    </div>
  `;
}
```
We can see the one-way binding that accepts the `primitiveValue`. In the template, this is printed using `{{"{{$ctrl.primitiveValue"}}}}`.

Let's take a look at our 'inner' component's controller.

```typescript
class InnerController {
  private onChangesCalledCounter: number = 0;
  public $onChanges(changes: any){
      this.onChangesCalledCounter++;
      console.log(changes);
  }
}
```
Notice the `$onChanges` definition in the controller. Every time it is executed, the `onChangesCalledCounter`, that is printed in the template as well, is incremented, and the changed object is logged to the console.

The whole thing is wired together as follows.

```typescript
angular.module("app", [])
  .component("outer",new OuterComponent())
  .component("inner", new InnerComponent());
```
Note that this uses the new `.component` method introduced in Angular 1.5. This is a kind of directive that always has an isolate scope and has it's `controllerAs` syntax set to `$ctrl` by default. Todd Motto has written an excellent [blog post](https://toddmotto.com/exploring-the-angular-1-5-component-method/) on the subject.

Let's try input some text in the input box and see what it does:
<iframe style="width: 100%; height: 250px" src="https://embed.plnkr.co/ZB6CLEXuRMrEnTIjUpo7/" frameborder="0" allowfullscren="allowfullscren"></iframe>

As you might have expected, on every key input in the input box, the `$onChanges` function is invoked.
With what we've learned, this is quite logical. A key input means the `primitiveValue` in the outer component has changed and a digest cycle is started. This will see that the `$watch` expression that is setup for the `'<'` binding is dirty and register the `$onChanges` callback to be notified during the `$$postDigest` phase.
Neat right!

###### Non primitive values
Let's kick it up a notch! Next is an example where we bind an array from the 'outer' to the 'inner' component. This is the [plnkr](http://plnkr.co/edit/iUy6m9s49m6nXLEqIK1P?p=preview).
Let's look at the 'outer' component's controller.

```typescript
class OuterController {
  private arrayValue: Array<String>;
  private fruits: Array<String> = ["apple", "banana", "grape"];
  private vegetables: Array<String> = ["brocolli", "aubergine", "avocado"];

  public $onInit(): void {
    this.arrayValue = this.fruits;
  }

  // In a real case scenario, this could be a network call that refreshes with new data
  public switchArrayValue(): void {
    if(this.arrayValue === this.fruits){
      this.arrayValue = this.vegetables;
    } else {
      this.arrayValue = this.fruits;
    }
  }
}
```

A controller is introduced that initialises an array called `arrayValue` to the fruits array. It also contains a `switchArrayValue` function that simply switches the reference from the `arrayValue` between the fruits and vegetables.
In our template, we've bounded the array to the 'inner' component and added a button to call the `switchArrayValue` function.

The 'inner' component only changed in accepting an array instead of a primitive value.

Let's try to click the button and see what it does:
<iframe style="width: 100%; height: 250px" src="https://embed.plnkr.co/iUy6m9s49m6nXLEqIK1P/" frameborder="0" allowfullscren="allowfullscren"></iframe>

As you would've expected, it does exactly the same thing as the previous example for the same reasons.

###### Non primitive values pitfall!

Next, let's try to add an element to the array in the 'outer' component and see what happens.
To do this, we must update the 'outer' component.

```typescript
class OuterController {
  private fruits: Array<String> = ["apple", "banana"];

  public addElement(): void {
    this.fruits.push("grape");
  }
}
```

We updated the controller to work with a fruit array and provide a method that adds an element to the array. In the template we just call this function when the button is clicked and bind the fruits array into our 'inner' component.

What do you think will happen next? You've figured it out? Let's give it a go!

<iframe style="width: 100%; height: 250px" src="https://embed.plnkr.co/Godmmy48YR5jv8O5fInI/" frameborder="0" allowfullscren="allowfullscren"></iframe>

Hmm, we see that the array in our 'outer' and our 'inner' component is updated. BUT our `$onChanges` callback is not fired. At first this might seem strange, but actually, it isn't.
To explain why, we must look at the definition of the `$watch` function in Angular.

```javascript
$watch: function(
    watchExp, listener, objectEquality, prettyPrintExpression
    )
```
Take a look at the third argument: **objectEquality**. This tells Angular whether to check the watch expression via reference or object equality. We looked at the way the watch expression is setup in the `recordChanges()` method above. No parameter is passed as third argument which means it defaults to comparison by reference.
In our test case above, we did not change the reference of the array but only added an element. This means our `$watch` expression will never be dirty! In that case the `$onChanges` callback is never triggered.
By now, you might wonder, then how the hell did our 'inner' component's array change. That's the thing. The 'outer' and 'inner' component share the same javascript reference to the fruits array. So even if you would push an element onto the array in the 'inner' component, it would also update the 'outer' component. Put's a whole new light onto 'one-way binding' doesn't it :).

###### Immutable data structures to the rescue!
You might wonder if the Angular team made an error of judgement when implementing it this way. In fact, they didn't. It was a deliberate choice to work with a reference-based comparison scheme. If you'd perform a deep compare on every object, your application's performance would be extremely poor.
So what can we do to get the expected behaviour for our `$onChanges` method. If we were to change the reference of our array every time we change something to the array, we would be getting exactly that. What we actually need are immutable data structures.
For the following example [plnkr](http://plnkr.co/edit/ANZnoxxe2K2pztlrPRfu?p=preview), I've chosen [ImmutableJS](https://facebook.github.io/immutable-js/) to demonstrate that we can get the expected behaviour with immutable data structures.

I've made a few small changes to the way the fruits array is created and updated:

```typescript
class OuterController {
  private fruits: List<String> = Immutable.List.of("apple" "banana");

  public addElement(): void {
      this.fruits = this.fruits.push("grape");
  }
}
```
First of all, we initialise our list using ImmutableJS. The push method that is available on the returned object, of type `List`, doesn't add an element to the existing datatype. Instead it returns a new instance of the List type with the element added. This way, we change our reference **and** add an element.

Let's test again to see if it works as expected.
<iframe style="width: 100%; height: 250px" src="https://embed.plnkr.co/ANZnoxxe2K2pztlrPRfu/" frameborder="0" allowfullscren="allowfullscren"></iframe>

And it does!

#### Conclusion

The lifecycle hook provides us with some great functionality. But be cautious. It's only triggered if we use **primitives** or **change the reference of the javascript object bound into the component**. That's why I would recommend to always use immutable data structures throughout your bindings.









