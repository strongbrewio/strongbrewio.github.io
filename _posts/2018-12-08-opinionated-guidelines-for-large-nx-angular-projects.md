---
layout: post
title: Opinionated guidelines for large nx angular projects
published: true
description: 
author: brechtbilliet
comments: true
date: 2018-12-08
subclass: 'post'
categories: 'brechtbilliet'
disqus: true
tags: Angular nx architecture
cover: 'assets/images/cover/cover2.jpg'
---

## About this article

This article contains a set of opinionated guidelines when it comes to building monorepos with [Nx](https://nrwl.io/nx).
I wrote this article because when I used Nx in the beginning, I struggled a lot with how to structure my workspace, and I've hit quite a few walls. That being said, I've been using it for a while now, and I finally have the feeling that I've reached a structure where I feel comfortable with. 

## A word about Nx

Nx is a thin layer on top of the [Angular](https://angular.io) CLI that helps us with structuring large applications in the form of monorepos.
A monorepo contains only one [Nx workspace](https://nrwl.io/nx/guide-nx-workspace) that can contain multiple apps and multiple libs (we will refer to apps and libs as Nx projects). An app is a deployable unit and a lib is meant to contain the actual logic that can be shared across the workspace. Nx is actively being developed and updated along with Angular by the amazing people of [Nrwl](https://nrwl.io/).

Nx is especially valuable when managing **big Angular applications** that have a lot of shared functionality, but even in smaller projects it can help organize your approach.

At StrongBrew we are using this technology for a bunch of our clients and even though Nx is already pretty opinionated, I decided to write down some best practices and guidelines that I try to take in consideration. 

The rules and guidelines written down in this article **might work for you**, and should in no circumstances be treated as the *ultimate truth*. Best practices and guidelines are mostly a matter of perception and personal preference. Nevertheless, I would love to share how I architect large Angular applications with Nx.

## Barrel files

When it comes to managing monorepos, barrel files are quite important.
A barrel file is a `index.ts` file that lives in the `src` directory of every Nx lib and is meant to expose logic to the rest of the workspace.

This file is really important when you understand one of the big potential risks of organizing code in monorepos - overexposure of implementation details.

With code being located right next to each other, it can be easy to import code with deeply nested relative paths and include things that the original author of the code never intended to be used outside of their specific context.

The Nx lib's `index.ts` file allows each lib to define its effective public API - only symbols which are explicitly exported from this file should be eligible for consumption in other parts of the workspace.

Let's say that we have a `@strongbrew/users` lib which exposes a `UserService`...
This is what the barrel file from  `@strongbrew/users` might look like.

```typescript
// libs/users/src/index.ts
export * from './lib/services/user.service';
```

Although this might seem pretty straight forward, let's go over a few best-practices...

### Don't ever import a lib from a relative path

When we want to import `UserService` inside another lib or app, we want to import it from `@strongbrew/users`. This is way cleaner then importing it from a relative path like `../../../users/lib/src/index.ts` and helps protect us from the overexposure problem described above. Nx also provides a linting rule out of the box to make sure that you are respecting a lib's API and not doing deep imports.

Nx uses TypeScript [path mapping](https://www.typescriptlang.org/docs/handbook/module-resolution.html#path-mapping) to map these module names to the correct barrel files.

```json
"paths": {
    "@strongbrew/users": {
        "libs/users/src/index.ts"
    }
}
```

### Only 1 barrel file per lib

It's a known fact that barrel files might become big, but it gives us a central place of handling all the exports + it drastically reduces the chance of getting circular reference errors. Therefore, a lib should only contain 1 single barrel file.

### Never let a lib import from its own Barrel file

The TypeScript modules within a particular lib should not care what functionality that lib exposes, so it shouldn't use its own barrel file at any point.

If a module imports something from its own barrel file, it almost always results in circular reference errors. Therefore, imports from inside of the module should use relative path imports.

## Structuring the workspace

When using Nx, we might already get pushed in an opinionated way of working, which is great. But how are we going to structure the workspace itself? For instance, looking into a directory of 100 libs inside of a libs directory might not really be pragmatic to work with...

### Structuring apps

#### An app should be an empty shell

Apps are deployable units that wire the different pieces of an application together. These apps are nearly empty shells that use libs to build an entire application. Therefore **an app holds almost no logic** and mostly uses lazy loading to load feature libs. Some feature libs can be seen as microfrontends.
That being said, our apps mostly not completely empty. In general they also contain:
- The general layout (composed out of components from 'ui-kit')
- Routing


#### Keep the apps directory as flat as possible

Chances are small that our monorepo will contain 100+ apps and even if it does, chances are small that we can divide these apps into categories.

#### Apps should not import from other apps

Although it might seem obvious, let's mention it anyway... Shared logic should always live inside of libs, an not inside of apps. Apps are specific deployment targets.

### Structuring libs

Here comes the opinionated part, lets check how we can structure the libs inside of our workspace.

The directory structure of our workspace might look like this:
- `apps`
- `libs`
    - `feature`
        - `api`
            - `foo`
            - ...
        - `lazy`
            - `bar`
            - ...
        - `shared`
            - `baz`
            - ...
    - `ui-kit`
    - `utils`

- A feature contains logic specific to a certain domain, like managing users or performing authentication. 
- On the other hand, the `utils` lib contains logic that doesn't have anything to do with any domain, E.g: HTTP interceptors, shared RxJS operators, a service that handles notifications etc... Consider it a toolkit for your workspace.

In the next few sections we are going to cover the 3 types of feature libs, the `ui-kit` lib and the `utils` lib.

#### feature/api

This directory contains Nx libs with a very specific purpose:
- These libs contain **api logic** or **business logic** that needs to be shared.
- These libs contain the types of the REST responses. Let's call them **domain types**.
- If we want to work with models, or dto's, these would also live here.

Having a specific api lib is very handy when developing in a microservices platform. Every microservice would have its own **api lib** that can be used throughout the entire monorepo.

Another common use-case is that feature libs tend to use domain types from other feature libs. By extracting these domain types in to api libs, we solve that problem. That way, these domain types can be shared across different places inside of the monorepo.

#### feature/lazy

This directory contains all feature libs that can be lazyloaded. To make sure these libs can be lazy-loaded, they should expose an `NgModule` in the barrel file and are loaded as such:

```typescript
RouterModule.forRoot([
    {
        path: 'users',
        loadChildren: '@strongbrew/feature/lazy/users'
    }
])
```

One of the advantages is that these modules can be loaded on demand or even preloaded upfront. The biggest advantage though is that these modules are completely standalone, and don't share anything with the rest of the workspace. This means they have nothing inside of their barrel file, other then the `NgModule` being exported. 
**Lazy loaded modules can never share logic with the workspace**
If we feel that a lazyloaded module needs to export something, we should extract that logic into a separate `feature/shared` or `feature/api` lib.

When a `feature/lazy` module needs to perform XHR calls it should delegate it to a `feature/api` lib. Therefore a `feature/lazy` lib should never contain api logic.

When using a statemanagement library like [ngrx/store](https://github.com/ngrx/platform), `feature/lazy` libs would contain their own reducers and use `store.forFeature()` to attach these to the `store` instance. This would result in lazy-loaded reducers.

#### feature/shared

Not every feature can be lazyloaded. Think about feature logic that needs to be shared for instance. In that case we would create an Nx lib that lives inside of the `feature/shared` directory.

When a `feature/shared` module needs to perform XHR calls it should delegate it to a `feature/api` lib. Therefore the `feature/lazy` lib should not contain api logic.  

#### ui-kit

This lib contains all the shared presentational components that can be used in different applications. Think about dropdowns, datepickers and empty modals. A `user-detail` component for instance does NOT belong here. A monorepo can contain multiple `ui-kit` libraries. We should name them according to its purpose. E.g `ui-kit-mobile` is a common use case. 

The Ui-kit module contains an `ngModule` since we need it to declare and export our components/directives. The barrel file generally only exposes the `ngModule` since this is the vessel used to export the functionality.

However, a `ui-kit` might also export certain types in its barrel file... Like `DatepickerConfiguration` or other `ui-kit` specific types.

#### utils

This lib can contain all kinds of utilities. It could contain shared interceptors, guards, services and custom RxJS operators. Think about it als a framework toolbox that could benefit any application. We will NOT use an `ngModule` here for tree-shaking purposes. 

A `utils` lib will not contain any components. But if it contains pipes or directives we might need an `ngModule` for that.

When your workspace is small, one single `utils` lib might suffice, but it could become a good idea to split these up when the `utils` lib gets to big.

After splitting up, the directory structure of our workspace might look like this:
- `apps`
- `libs`
    - `feature`
        - `api`
            - `foo`
            - ...
        - `lazy`
            - `bar`
            - ...
        - `shared`
            - `baz`
            - ...
    - `ui-kit`
    - `utils`
        - `rxjs-operators`
        - `forms`
        - `http`

Do note, that the `forms` lib would not contain any forms or forms configuration, but it would contain general form logic that can be shared across the workspace.

### Prefixing libs

Because of the fact that `selector` names for components and directives should be unique, prefixing them in a monorepo is quite important. 
Since every project in the `angular.json` file has a `prefix` property, we could set that prefix for every project.

Let's assume that we need a `feature/shared` lib called `messages` then we could generate that lib by running ```ng g lib messages --prefix sh-mes``` for instance. 
`sh-mes` would be the prefix and if we create a message component in this lib it would have the selector: `sh-mes-message`.

## Linting and tags

One thing that is absolutely critical when managing a monorepo is being able to determine, categorize, and constrain/run commands based on a dependency graph.

Nx determines the dependency graph for us out of the box, it infers it by statically analyzing our TypeScript import and export statements (as well as a few other things specific to the Angular CLI).

It has no way of automatically categorizing the dependency graph for it, because that is up to our subjective judgement, but it does provide helpers to make it easy.

Nx provides us with the ability to add tags to the different libs and apps and apply [tslint](https://palantir.github.io/tslint/) rules to make sure we can't import whatever we want wherever we want.	Nx provides us with the ability to add tags to the different libs and apps and apply [tslint](https://palantir.github.io/tslint/) rules to make sure we can't import whatever we want wherever we want (potentially leading to circular references and other problems (broken lazyloading, etc...)).

Tags can be added to projects in the `nx.json` file of the root directory.
Tags can be determined in numerous ways. Some of us might like a tag per team, other might like it per domain. 

I like to have tags for every lib type. It might be opinionated but it works fine for me (again, that's a matter of personal preference)

We define 5 types of tags:

- `app`: This tag is added to all the apps
- `shared`: This tag is added to `uikit` and `utils` libs
- `feature:lazy`: This tag is added to `feature/lazy` libs
- `feature:shared`: This tag is added to `feature/shared` libs
- `feature:api`: This tag is added to `feature/api` libs

The rules could be the same for every workspace that we will create in the future:

- Projects with the type `tag` can only depend on projects with the tags: `shared` or `feature:shared`.
- Projects with the type `shared` can only depend on projects with the tags: `shared` (we don't want to import domain specific logic in there do we?)
- Projects with the type `feature:lazy` can only depend on projects with the tags: `shared`, `feature:shared` and `feature:api`.
- Projects with the type: `feature:shared` can only depend on projects with the tags: `shared` and `feature:api`.
- Projects with the type: `feature:api` can only depend on projects with the tags: `feature:api` and `shared`. (we never want to load `feature:shared` into an feature/api lib right?)

### Configuring tslint

To configure the tslint we have to use the `nx-enforce-module-boundaries` rule from tslint. If you like the rules defined above, you can just copy-paste the module boundaries defined below right in your `tslint.json` file that lives in the root directory.

```json
"nx-enforce-module-boundaries": [
  true,
    {
      "allow": [],
      "depConstraints": [
      {
        "sourceTag": "app",
        "onlyDependOnLibsWithTags": ["shared", "feature:shared"]
      },
      {
        "sourceTag": "shared",
        "onlyDependOnLibsWithTags": ["shared"]
      },
      {
        "sourceTag": "feature:lazy",
        "onlyDependOnLibsWithTags": [
          "shared",
          "feature:shared",
          "feature:api"
        ]
      },
      {
        "sourceTag": "feature:api",
        "onlyDependOnLibsWithTags": ["feature:api", "shared"]
      },
      {
        "sourceTag": "feature:shared",
        "onlyDependOnLibsWithTags": ["shared", "feature:api"]
      }
      ]
    }
]
```

This tslint config will ensure that the rules defined above are mandatory.

## Is this structure the only way?

No, not at all, this would work perfectly for a monorepo with 5 applications. But if we are thinking about organisation wide monorepos, it might be a good idea to
combine features app per app. In that case we would have something like:
- `apps`
- `libs`
    - `app1`
        - `api`
            - `foo`
            - ...
        - `lazy`
            - `bar`
            - ...
        - `shared`
            - `baz`
            - ...
    - `app2`
        - `api`
            - `foo`
            - ...
        - `lazy`
            - `bar`
            - ...
        - `shared`
            - `baz`
            - ...
    - `ui-kit`
    - `utils`
        - `rxjs-operators`
        - `forms`
        - `http`

## How to share code organisation wide?

Although an organisation wide monorepo has great benefits, they might be good reasons not to do it. These could be any combination of technical, cultural, legal or other reasons.

### Scenario A

Our company has 10 angular projects that are actively developed and share a lot of code, but also had 5 legacy projects where there is no budget to bump them to the new Angular versions and so on. There might be a few [Vue.js](https://vuejs.org/) or [React](https://reactjs.org/) living there as well. It might be more trouble than it is worth for your organization to manage that complexity within one big workspace. In that case we could have a workspace for the non-legacy angular projects, and that workspace would gladly welcome new projects in the future.

### Scenario B

Our company sells custom software to different clients. Every client wants its own custom look and feel, which a lot of custom logic, but we don't want to reinvent the wheel every time.

In that case, we could create an Nx worspace for every client, and have one common toolkit that contains shared logic. That toolkit would live in its own monorepo and be published as an Angular package.

## Conclusion

I hope we learned something today. How we structure our workspaces is completely up to us, and we should use something that works for us, not just pick whatever you read in some blog article ;-). If this structure doesn't make sense to you, that's perfectly fine... And I would love to hear your thoughts about this approach.

## Special thanks to

Thanks to the people that have reviewed the article and gave great input!
I couldn't have done it without you!

- [@beeman_nl](https://twitter.com/beeman_nl)
- [@MrJamesHenry](https://twitter.com/mrjameshenry)
