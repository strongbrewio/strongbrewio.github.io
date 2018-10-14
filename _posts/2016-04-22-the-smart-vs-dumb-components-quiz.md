---
layout: post
cover: false
navigation: True
title: The smart vs dumb component quiz
date:   2016-04-22
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
author: kwintenp
cover: 'assets/images/cover/cover2.jpg'
---
If you follow the blogs of some of the more well known SPA guru's, you will most definitely have heard of the concept of `smart` and `dumb` components. If you haven't, here's a listing of some of those blogposts:

- <a href="http://teropa.info/blog/2016/02/22/dumb-components-and-visual-feedback-in-angular-apps.html" target="_blank">Tero Parviainen: Dumb Components and Visual Feedback in Angular Apps</a>

- <a href="https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.wnlz25kho" target="_blank">Dan Abramov: Presentational and Container Components</a>

- <a href="https://toddmotto.com/stateless-react-components/" target="_blank">Todd Motto: Stateless React components</a>


**Note: These articles often use different names for the same concept. For the sake of it, I'll be using `smart` and `dumb` to define the components.**

Their even was a talk on <a href="https://www.youtube.com/watch?v=WfRmhYgwIho" target="_blank">NG-NL</a> by Shai Reznik on the topic, which was hilarious by the way.

What's noticeable is that the concept is applicable to different technologies. It isn't something that only applies to Angular or React. It's applicable to every SPA technology out there, making it even more interesting to really understand what it means.

Even if you've read every blogpost and watched the youtube video, putting the theory into practice isn't as easy as you might expect. If you think you're really getting what is meant by `smart` and `dumb` components, you can try and answer the following questions my good friend <a href="https://twitter.com/brechtbilliet" target="_blank">Brecht Billiet</a> has came up with. I've added explanations and examples where possible.

#### Question Time
1: Is a `dumb` component reusable?
2: Should a `dumb` component know about the application's state?
3: Should a `dumb` component do data fetching?
4: Should a `dumb` component redirect to another route?
5: Should a `dumb` component have DI in his constructor?
6: Should a `dumb` component dispatch events?
7: Should a `dumb` component contain logic?
8: Is a component that contains child elements a `smart`?

#### Answers

###### 1: Is a `dumb` component reusable?
Of course it is! That's the whole point. `Dumb` components have a clear API with inputs and outputs. This makes it easy to reuse the component.

###### 2: Should a `dumb` component know about the application's state?
It should **not** know anything about application state. If your component knew something about the state of the application, it would lose it's ability to be reusable. But even more important, by using components that do not know anything about the state, you're reducing the complexity of these components and your SPA as whole. You don't have to think about the fact that these component might update your state which is a huge benefit.

###### 3: Should a `dumb` component do data fetching?
No it should not. If your `dumb` component is used to visualise data, you should pass the data as a parameter to your component.
If you're, for example, creating widgets that are being used across multiple applications, dumb components are the way to go. If you'd do the data fetching in the `dumb` component, you wouldn't be able to use it as a widget.

###### 4: Should a `dumb` component redirect to another route?
No. If you'd do this, your component knows about the application's state and is no long reusable across different applications, since the route changes will be different.

###### 5: Should a `dumb` component have DI in his constructor?
It depends but mostly no. In general, the component **must not** rely on DI.
A case where it might be acceptable is when using feature toggles to hide or show some parts.

```typescript
class SmartComponent {
  public template: string = `<dumb>></dumb>`;
}
class DumbComponent {
  public template: string =
              `<div>showing</div>
               <div ng-if="$ctrl.featureModel.isActive()">
                   not-showing
               </div>`;
  public controller: Function = DumbController;
}
class DumbController {
  public static $inject: Array<string> = ["featureModel"];
  constructor(public featureModel){}
}
class FeatureService{
  public isActive(): boolean {
    return false;
  }
}
```
Here we have an example where a `smart` component contains a `dumb` component. The `dumb` component relies on the `featureService` to know if it should show a `div` or not.
You don't want to pollute the API of the `dumb` component by passing a `show` variable since this might be a temporary intervention. Check here for the <a href="http://plnkr.co/edit/Iz6C7F5QAveS60l6y9wB?p=preview" target="_blank">plnkr</a>.
Another example where DI is an option in a dumb component is when you want to use a TranslationService. This does not update application state nor does is perform a route change or influence other components in your application. You can use DI in that component without the component losing the label of being 'dumb'.

###### 6: Should a `dumb` component dispatch events?
Yes! When using this paradigm, you'd most certainly want to use a unidirectional flow of data. If you only pass data from the top of your component tree to the bottom, you must have a way for the `dumb` components to notify it's parent component when something changes or needs to be done.

Check out this <a href="http://plnkr.co/edit/kG60UcrSWzwfRjANRzGT?p=preview" target="_blank">plnkr</a>.

```typescript
class SmartComponent {
  public template: string =
        `<dumb  text="$ctrl.text"
                on-change="$ctrl.onChange(text)">
         ></dumb>`;
  public controller: Function = SmartController;
}
class SmartController {
  public text: string = "initial value";
  public onChange(text: string): void {
    console.log(text);
  }
}
class DumbComponent {
  public bindings: any = {
    onChange: "&",
    text: "<"
  };
  public template: string =
              `<input ng-model="$ctrl.text">
              <button   type="button"
                        ng-click="$ctrl.onChange({text: $ctrl.text})">
              Call to smart component</button>`;
}
```
This example has a `smart` component that contains a `dumb` component. The `smart` component passes a text and a callback method to the `dumb`. When the `dumb` component has updated the property it uses the callback method to notify the `smart` component. The responsibility of the dumb component here is: 'Accept some data modify it and return it via the callback method'.

**Note: I really think you should be able to describe a dumb component's functionality in a single sentence. It's a nice trick in knowing if the component should be dumb or smart.**

###### 7: Should a `dumb` component contain logic?
It can. But it can only be logic that relates to the component itself. This logic is not business logic.
You could, for example, hold a boolean value whether something in the component is collapsed or not.

###### 8: Is a component that contains child elements a `smart`?
No. It is perfectly acceptable for a `dumb` component to contain other `dumb` components. Of course, a `dumb` component cannot contain a `smart` component.

EDIT: As [@areksliwa](https://disqus.com/by/ArekSliwa/) pointed out in the comments, when you have a layout component where some content is being projected into via content projection, it can happen that a dumb component (the layout component) contains a smart component. This can happen for a tab(s) component for example.


Can you think of some more questions? Please let me know :).

