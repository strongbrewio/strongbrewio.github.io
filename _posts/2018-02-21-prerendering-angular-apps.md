---
layout: post
title: Prerendering angular applications
published: false
author: brechtbilliet
description: 
comments: true
---

## Foreword

There are several ways of optimizing Angular applications. For instance, we could compile them [ahead-of-time](https://angular.io/guide/aot-compiler) through AOT-compilation.
We could use service-workers to optimize caching. And there are plenty of other PWA (progressive web-app) features as well that can increase the quality of our Angular applications.

However, there are a still a few problems that these optimizations won't fix:
- SEO (Search engine optimization): At the time of writing the application is harder to index by search engines because the content isn't available on load time. Therefore the application is likely to fail on several SEO requirements.
- Initial page load: Since the application still needs to be bootstrapped after the pageload, there is an initial waiting time untill the user can use the application. This results in lesser user experience.

These two problems can be fixed by SSR (Server side rendering). SSR takes the best of both worlds: We still have an SPA (single page application) that gives us the native feeling but the page is still rendered instantly. So the application is rich and fast at the same time!

To optimize our [StrongBrew](https://strongbrew.io) website we started by using this approach. It was pretty great when we ran it locally, however the StrongBrew website is hosted on [Firebase](https://firebase.google.com) and the SSR part was hosted by Firebase functions. We really love Firebase and everything it stands for, but for us, SSR on Firebase functions was just too slow. Sometimes it took 4 seconds to serve the content...

Since the loading time of a website is very important if we want to keep our visitors, we had to find another way to serve the content in a more effective manner. The key to SSR is Angular Universal, and so it turns out we can use just that to prerender all the html at build-time instead of at run-time. 
That way we would run generated static html files which is insanely fast, and when the javascript bundles are loaded the browser would take over.
This also uses the best of both worlds: Ultra fast loading time + we don't need to give up a rich experience.
The result went from several seconds to 30 miliseconds.

This is a super fast and super effective improvement but it has one very important limitation.
**It's not possible to prerender dynamic content**. The data in the StrongBrew website isn't fetched by AJAX calls (at least not the data that has to be indexed), it rather works with simple webpack imports of JSON files. These are inserted at build time :)

This does not mean that loading content dynamically isn't possible anymore, it just won't get prerendered.

## Let's dive in

Enough chit chat! Let's dive into some code!
I've created this [github repository](https://github.com/strongbrewio/prerender-angular-example) just for you! It's a simple website with a few pages and the build system doesn't know how to prerender yet.
Checkout the branch `runtime` by running the command `git checkout runtime`. When running `npm i && npm run start` the bash should install all the NPM dependencies and host the application on http://localhost:4000, just like any default Angular-CLI application.

### Installing the dependencies

Great! We have a running website, but nothing is prerendered yet.
The first thing we need to do is installing @angular/platform-server by running
`npm i @angular/platform-server -D` This is the most important bundle for SSR. This module contains the basics to run the Angular application on the server.

Next we need to update the app.module to enable server transition. This will make sure that Angular takes over on the frontend when the javascript is loaded.

### Applying the server transition

```typescript
// src/app/app.module.ts
@NgModule({
  ...
  imports: [
    BrowserModule.withServerTransition(
        // this is just the name of our application
        { appId: 'prerender-angular-example' }
    ),
    ...
  ],
  ...
})
export class AppModule { }
```

### Creating the prerender module

Next we need to create a specific prerender module that will use the AppModule we just adjusted. Let's create an app.prerender.module.
It's important that we tell this module which component it has to bootstrap.

```typescript
// src/app/app.prerender.module.ts
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

### Defining a prerender entrypoint

To use the SSR logic at build time we need to have a specific bundle. Since the main.ts file is used to bootstrap the application for the browser we also need a main.prerender.ts file that will be used to create the prerender bundle.
Let's create that file shall we?

```typescript
// src/app/main.prerender.ts
import { enableProdMode } from '@angular/core';
export { AppPrerenderModule } from './app/app.prerender.module';

enableProdMode();
```
### A prerender tsconfig.json

We are almost there I promise, but we need a few more things. We need a specific tsconfig file that compiles the bundle to something that the node server can read. It's key that compiler compiles to a commonjs package. So we need to create a `tsconfig.prerender.json` file

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

### Letting Angular Cli know

In the apps entry of the `angular-cli.json` file, we need to add a new app where we refer to the `main.prerender.ts` file and the `tsconfig.prerender.json` file. The app section should look like this:

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

## Generating the bundle

Update the package json so it builds both the normal package and the server package. Set the output-hashing to none so that the build generates a clean `main.bundle.js`
```
    "build": "ng build --prod && ng build --prod --app prerender --output-hashing=none",
```
when running npm run build the following files should be created:
- dist (this contains the normal build)
- `dist-prerender/main.bundle.js`

## Generating the static files

We have just created the `main.bundle.js`. this file could be used to perform server side rendering. However we don't want to do server side rendering in this case, we want to prerender the html at build time. To do that we need a script that will complete the following steps.

- Create an array with routes (you could automate this if you want)
- Loop over that array and for every entry: 
  - create a folder in the dist map with that route name 
  - use the ```main.bundle.js``` to render the html and store that html as an ```ìndex.html``` in the folder we just created.

Let's call that script prerender.ts. Since I'm a typescript enthusiast I want to create the prerender script in typescript and use ts-node to run it.
We can start with creating an empty prerender.ts file in the root folder and installing ts-node with ```npm i -D ts-node```

Now we can update the scripts section of the package.json so that the render function is called when the build is completed:

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

Right now everything is bootstrapped to finish the whole cycle. The only thing that rests us is implementing the prerender functionality.
Let's go!
  ### Completing the prerender.ts file