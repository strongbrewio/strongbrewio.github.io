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

But what if it's only a single component that cannot be rendered when the user does not have the proper role? Angular does not provide something out of the box for this. Luckily this is something we can pretty easily implement using directives. 

<iframe src="https://player.vimeo.com/video/251380600" width="640" height="360" frameborder="0" webkitallowfullscreen mozallowfullscreen allowfullscreen></iframe>
<p><a href="https://vimeo.com/251380600">Displaying components based on the role of a user</a> from <a href="https://vimeo.com/user79085465">Kwinten Pisman</a> on <a href="https://vimeo.com">Vimeo</a>.</p>

### Defining what we want
We want to create a directive that accepts a role. This is the role the user must have to see this specific component.
It might look something like this:

```html
<app-normal-users-can-view *appHasRole="'user'">
</app-normal-users-can-view>
```


### Defining how it should work

Lets define the characteristics for our directive:
* If the user has the proper role, the component should be shown
* If the user does not have the proper role, the component shouldn't be added to the DOM

If you think about it, that's pretty much what an `*ngIf` directive does. Based on a certain condition, a component or native element must be added or removed from the DOM.

### What does our directive need to make it work
Our directive needs several things to make it work properly. First of all, we need a reference to the element that needs to be added to the DOM. Secondly, we need a place where we can insert the element to the DOM if needed. And lastly, we need to know the roles the user has.

#### Getting a reference to the element that needs to be added

To get a reference to the element we need to add, we can use a 'template directive' (or structural directive). We can make any directive a 'template directive' by prefixing the directive with an asterisk when we use it. This is syntactic sugar for the angular compiler. If we do this, this is what the compiler actually sees:

```html
<!-- what we write -->
<app-normal-users-can-view *appHasRole="'user'">
</app-normal-users-can-view>

<!-- how the compiler interprets it -->
<ng-template [appHasRole]="'user'">
      <app-normal-users-can-view></app-normal-users-can-view>
</ng-template>
```

We can see that the compiler puts our directive onto an `ng-template`. This is ideal because it allows our directive to inject the `TemplateRef` containing the element we need to add (if the user has the role), which is what we want.

#### Finding the DOM entry point
To know where we can inject the `TemplateRef`, our directive can just inject the `ViewContainerRef`. This represents the container where we can attach one or more views. For more information on the `ViewContainerRef` you can read <a href="https://netbasal.com/angular-2-understanding-viewcontainerref-acc183f3b682" target="blank">this</a> post.

#### Knowing the roles the user has
To know the roles a certain user has, we could leverage a service (in the example code `rolesService`) that exposes a stream with all the roles the user has.

### The hasRole directive end result
Now that we have identified all the different things we need to properly implement our directve, we can start. See the comments for more information.

```typescript
@Directive({
  selector: '[appHasRole]'
})
export class HasRoleDirective implements OnInit, OnDestroy {
  // the role the user must have 
  @Input() appHasRole: string;

  stop$ = new Subject();

  isVisible = false;

  /**
   * @param {ViewContainerRef} viewContainerRef 
   * 	-- the location where we need to render the templateRef
   * @param {TemplateRef<any>} templateRef 
   *   -- the templateRef to be potentially rendered
   * @param {RolesService} rolesService 
   *   -- will give us access to the roles a user has
   */
  constructor(
    private viewContainerRef: ViewContainerRef,
    private templateRef: TemplateRef<any>,
    private rolesService: RolesService
  ) {}

  ngOnInit() {
    //  We subscribe to the roles$ to know the roles the user has
    this.rolesService.roles$.pipe(
    	takeUntil(this.stop$)
    ).subscribe(roles => {
      // If he doesn't have any roles, we clear the viewContainerRef
      if (!roles) {
        this.viewContainerRef.clear();
      }
      // If the user has the role needed to 
      // render this component we can add it
      if (roles.includes(this.appHasRole)) {
        // If it is already visible (which can happen if
        // his roles changed) we do not need to add it a second time
        if (!this.isVisible) {
          // We update the `isVisible` property and add the 
          // templateRef to the view using the 
          // 'createEmbeddedView' method of the viewContainerRef
          this.isVisible = true;
          this.viewContainerRef.createEmbeddedView(this.templateRef);
        }
      } else {
        // If the user does not have the role, 
        // we update the `isVisible` property and clear
        // the contents of the viewContainerRef
        this.isVisible = false;
        this.viewContainerRef.clear();
      }
    });
  }
  
  // Clear the subscription on destroy
  ngOnDestroy() {
    this.stop$.next();
  }
}

```
You can find the full source code <a href="https://github.com/KwintenP/display-or-hide-components-based-on-role" target="_blank">here</a>. You can play with a live example <a href="" target="_blank">here</a>. You can change the roles on top by clicking the checkboxes. If a role is granted, a new element is added to the DOM.

### Conclusion 
Leveraging a few somewhat more advanced concepts as 'template directives', `viewContainerRef` and `TemplateRef`, we were able to easily implement our own `*ngIf` like directive that works based on the roles with a limited amount of code.

**Note:** In this example I'm using roles. Applying a role based strategy for authorization poses some problems. There will always be cases where a user should have role 'X' but should also be able to see a portion of the functionality that users with role 'Y' see. In that case, you would have to create a new role, 'Z', that holds properties of 'X' and 'Y'. 

In growing applications, this will mean a lot of roles, that might be only used by a single person. With that in mind, it's always better to give the user certain 'rights'. If the user can see the 'User management' part of the application, he should have the 'right' 'user_mgmt' for example. Using 'rights' avoids the problem described above with the roles.