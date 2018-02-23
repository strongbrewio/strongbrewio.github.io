---
layout: post
title: Prerendering angular applications
published: false
author: brechtbilliet
description: 
comments: true
---

## Foreword

There are several ways of optimizing Angular applications. We could compile them ahead of time through AOT-compilation.
We could use service-workers to optimize caching. And there are plenty of other PWA (progressive web-app) features as well that can increase the quality of our app.

However, there are a still a few problems that these optimizations won't fix:
- SEO Search engine optimization: The application is harder to index by search engines because the content isn't available on load time. Therefore the application is likely to fail on several SEO requirements.
- Initial page load: Since the application still needs to be bootstrapped after the pageload, there is a waiting time untill the user can use the application

These two problems can be fixed by SSR (Server side rendering). SSR takes the best of both worlds: We still have a SPA (single page application) that gives us the native looking javascript app but the page is rendered instantly. So it's rich and fast at the same time!

For our [StrongBrew](https://strongbrew.io) website we tried to use SSR. It was pretty great when we ran it locally, however the StrongBrew website is hosted on Firebase and the SSR part was hosted by Firebase functions. We really love Firebase and everything it stands for, but SSR on Firebase functions was really slow. Sometimes it took 4 seconds to serve the content...

Since the loading time of a website is very important if we want to keep our visitors, we had to find another way to serve the content in a more effective manner. The key to SSR is Angular Universal, and so it turns out we can use just that to prerender all the html at build-time instead of at run-time. The result went from several seconds to 30 miliseconds.

This is a super fast and super effective improvement but it has one important limitation.
**It's not possible for apps with dynamic content**. The data in the StrongBrew website isn't fetched with AJAX calls (at least not the data that has to be indexed), it rather works with simple webpack imports of JSON files. These are inserted at build time :)

## Let's dive in

Enough chit chat! Let's dive into some code!
I've created this [github repository](https://github.com/strongbrewio/prerender-angular-example) just for you! It's a simple website with a few pages that doesn't know how to prerender yet.
Checkout the branch `runtime` by running the command `git checkout runtime`. When running `npm i && npm run start` the application should install all its NPM dependencies and get hosted on http://localhost:4000, just like any default Angular-CLI application

## Angular platform server
npm i @angular/platform-server -D

Update the app.module to enable server transition

```typescript
@NgModule({
  ...
  imports: [
    BrowserModule.withServerTransition(
        { appId: 'prerender-angular-example' }
    ),
    ...
  ],
  ...
})
export class AppModule { }
```

Create a app.prerender.module
```typescript
// src/app/app.server.module
import { NgModule } from '@angular/core';
import { ServerModule, ServerTransferStateModule } from '@angular/platform-server';

import { AppModule } from './app.module';
import { AppComponent } from './app.component';

@NgModule({
  imports: [
    AppModule,
    ServerModule,
    ServerTransferStateModule
  ],
  bootstrap: [AppComponent]
})
export class AppPrerenderModule {
}
```

Create a main.prerender.ts
```typescript
// src/app/app.server.module.ts
import { enableProdMode } from '@angular/core';
export { AppPrerenderModule } from './app/app.prerender.module';

enableProdMode();
```

Create a tsconfig.prerender.json
```json
/* src/tsconfig.prerender.json */
{
  "extends": "./tsconfig.app.json",
  "compilerOptions": {
    "outDir": "../out-tsc/prerender",
    /* node only understands commonjs for now*/
    "module": "commonjs"
  },
  "exclude": [
    "test.ts",
    "**/*.spec.ts"
  ],
  /* Additional informations to bootstrap Angular */
  "angularCompilerOptions": {
    "entryModule": "app/app.prerender.module#AppPrerenderModule"
  }
}

```

create an app in the angular-cli.json and refer to the main.server.ts and the tsconfig.server.json

```json
{
      "name": "prerender",
      "platform": "server",
      "root": "src",
      "outDir": "dist-prerender",
      "main": "main.prerender.ts",
      "tsconfig": "tsconfig.prerender.json",
      "environmentSource": "environments/environment.ts",
      "environments": {
        "dev": "environments/environment.ts",
        "prod": "environments/environment.prod.ts"
      }
    }
```

Update the package json so it builds both the normal package and the server package. Set the output-hashing to none so that the build generates a clean main.bundle.js
```
    "build": "ng build --prod && ng build --prod --app prerender --output-hashing=none",
```
when running npm run build the following files should be created:
- dist (this contains the normal build)
- dist-prerender/main.bundle.js

## Generating the static files
To generate the static files we need to create a prerender script. Since I'm a typescript enthusiast I want to create the prerender script in typescript and use ts-node to run it.

Let's start with creating an empty prerender.ts file in the root folder and installing ts-node with ```npm i -D ts-node```

Now we can update the scripts section of the package.json so that the render function is called when the build is completed:

UPDATE TO TYPESCRIPT 2.6.2
```json
 "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build --prod && ng build --prod --app prerender --output-hashing=none",
    "postbuild": "npm run render",
    "render": "ts-node prerender.ts",
    ...
  },
  ```

  ### Completing the prerender.ts file