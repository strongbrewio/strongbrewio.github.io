---
layout: post
cover: false
title: Type safe actions in reducers
date:   2016-11-21
subclass: 'post'
categories: 'kwintenp'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: Redux
cover: 'assets/images/cover/cover4.jpg'
---
I've been using TypeScript and Redux for a while now. One thing that's been bothering me from day one is the lack of typing on actions, or so I thought. Until the following twitter conversation last week caught my eye.

![twitter](https://www.dropbox.com/s/28omnkkkn1o2rm1/Screenshot%202016-11-16%2016.45.43.png?raw=1)

It's about an exhaustive switch statement in flowtype and the option to do the same in TypeScript. But the real interesting part for me was the link from Mike Ryan. He pointed out they used some kind of pattern in the ngrx example app. Having never checked it out before, I decided to do so, and I found this cool idea to use classes for your actions.

I loved it so much, I decided to blog about it :). All credits to Mike Ryan of course who came up with the idea.

### My previous situation
This is what my old code looked like (see comments).

```typescript
// Create string constants for the action types
const SET_ID: string = "SET_ID";
const REMOVE_ID: string = "REMOVE_ID";

// Create action creators for every action
function setId(id): Action {
	return { type: SET_ID, payload: { id } };
}

function removeId(): Action {
	return { type: REMOVE_ID };
}

function test(state: string = "", action: Action): string {
    // switch on the action type
	switch (action.type) {
		case SET_ID:
		     // have absolutely no type safety on the
		     // payload here since payload is
		     // defined as 'any'
		     return action.payload.id;
		case REMOVE_ID:
	         return "";
		default:
		     return state;
	}
}

```
While this is perfectly valid code, it doesn't provide me with any code completion or type safety regarding the payload.
Just check out this <a href="http://bit.ly/2fVxE7C" target="_blank">TypeScript playground</a> example and try to change the `action.payload.id` into `action.payload.whatever`. You will see no compile errors.

Let's see how this can be improved.

### Use classes to define actions
In the following code snippet I used classes for actions instead of action creators. These classes extend from the Action interface. This means, every class will have the `type` property.
I also created a new <a href="https://www.typescriptlang.org/docs/handbook/advanced-types.html#union-types" target="blank">Union type</a> called `Actions` which combines all the possible action classes.

Our switch statement works on the common denominator between all our actions, being the `type` property. This way the TypeScript compiler can
know that, if the type is for example `"SET_ID"`, the only possible class in that specific 'case' part of the switch statement is the `SetId` class. It can then use the type information in that class to determine what the payload looks like. Check the code below if this is unclear.

This is a concept called <a href="https://www.typescriptlang.org/docs/handbook/advanced-types.html#discriminated-unions" target="_blank">Discriminated Unions</a>.

```typescript
// Instead of using action creators, classes are used.
class SetId implements Action {
	type: "SET_ID" = "SET_ID";
	// here we declare the type of the payload for the
	// SetId class to be an object with a property 'id'
	payload: { id: string };

	public constructor(id: string) {
		this.payload = { id };
	}
}

class RemoveId implements Action {
	type: "REMOVE_ID" = "REMOVE_ID";

	public constructor() { }
}

// Create a union type that contains all the possible actions.
type Actions = SetId | RemoveId;

function test(state: string = "", action: Actions): string {
    // The switch case statements use discriminated unions
	switch (action.type) {
		case "SET_ID":
		    // The compiler knows this can only be the
		    // class SetId so it can use the type
		    // information in that class
		    // to know the payload has an id property.
			return action.payload.id;
		case "REMOVE_ID":
			return "";
		default:
			return state;
	}
}

```

You can try this <a href="http://bit.ly/2fXYiPB" target="_blank">TypeScript playground example</a>. If you remove the id property in the switch statement, you'll see that you have autocompletion

![autocomplete](https://www.dropbox.com/s/1s4zyh01xbp8g2a/Screenshot%202016-11-16%2020.31.03.png?raw=1)

and type safety!

![type safety](https://www.dropbox.com/s/6wtt9oupqr8290z/Screenshot%202016-11-16%2020.45.46.png?raw=1)


Just try to change the property `id` into `whatever`, you'll get a compilation error. You can even click on the `id` property and directly be redirected to the `SetId` class.

Awesome right!

### The finishing touch
The way the type property in the classes were defined before, are a little strange.

```typescript
type: "REMOVE_ID";
```
This is actually called a <a href="https://www.typescriptlang.org/docs/handbook/advanced-types.html#string-literal-types" target="_blank">String literal type</a>.
There's a better way to do this using a utility method that coerces a string you pass to it to a string literal type. It also remembers every action type you've passed to it to avoid duplicates in your app.

```typescript
let typeCache: { [label: string]: boolean } = {};
export function type<T>(label: T | ''): T {
  // this actually checks whether your action type
  // name is unique!
  if (typeCache[<string>label]) {
    throw new Error(`Action type "${label}" is not unqiue"`);
  }

  typeCache[<string>label] = true;

  return <T>label;
}
```

Using this function, you can declare your action types like this:

```typescript
export const ActionTypes = {
	SET_ID: type<"SET_ID">("SET_ID"),
	REMOVE_ID: type<"REMOVE_ID">("REMOVE_ID")
}
```

and use them everywhere like this:

```typescript
type = ActionTypes.SET_ID;
// or
case ActionTypes.SET_ID:
```

Checkout the finished <a href="http://bit.ly/2m7nG7S" target="blank">TypeScript playground example</a>. It's basically the same as the previous one, but cleaner.

### Conclusion
Using some of TypeScript 2's powerful typing system, you can make the actions in your reducers type safe with little effort.

**Note:** Thanks to <a href="https://twitter.com/PascalPrecht" target="_blank">Pascal Precht</a>, <a href="https://twitter.com/toddmotto" target="_blank">Todd Motto</a>, <a href="https://twitter.com/basarat" target="_blank">Basarat</a> and <a href="https://twitter.com/SamVerschueren" target="_blank">Sam Verschueren</a> for reviewing!

