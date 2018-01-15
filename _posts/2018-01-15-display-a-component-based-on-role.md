---
title: Displaying components based on the role of a user
published: false
author: kwintenp
description: This article demonstrates how we can show or hide certain parts of the application if a user has the right to see it or not
layout: post
navigation: True
date:   2018-01-15
tags: Angular
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover4.jpg'
---
At some moment in time, almost every application will have certain parts that need to be restricted to users with the proper roles. When we need to protect a certain route from unauthorized access, Angular provides us with a guard.

But what if it is only a single component that cannot be rendered when the user does not have the proper role? Angular does not provide something out of the box for this. Luckily this is something we can pretty easily implement using directives. 

// insert video

### Defining what we want
We want to be able to apply a directive to any element that accepts a role the user must have as input. Visualised it looks like this:

```html
<app-normal-users-can-view *appHasRole="'user'">
</app-normal-users-can-view>
```


### Defining how it should work

Lets define the characteristics for our directive:

* If the user has the proper role, the component should be shown
* If the user does not have the proper role, the component shouldn't be added to the DOM

If you think about it, that's pretty much what an `*ngIf` directive does. Based on an 

As you can see, we are prefixing the directive with an asterisk. By prefixing it, we made it a template directive. This is syntactic sugar for the Angular compiler. 
This is what the compiler sees:

```html
<!-- from -->
<app-normal-users-can-view *appHasRole="'user'">
</app-normal-users-can-view>

<!-- to -->
<ng-template [appHasRole]="'user'">
      <app-normal-users-can-view></app-normal-users-can-view>
</ng-template>
```