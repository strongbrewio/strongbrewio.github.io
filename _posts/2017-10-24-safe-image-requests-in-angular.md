---		
layout: post		
title: Safe image requests in angular
published: true
author: brechtbilliet
description: Since the browser just loads this image through a normal HTTP call, how is the server supposed to know if user A or user B is trying to consume an image? The server needs some kind of authorization to be passed to it. This article explains how to add security to images in Angular.
comments: true
date: 2017-10-24
subclass: 'post'
categories: 'casper'
disqus: true
tags: Angular
---
We all know how to load images in a web-application right? We would just make use of the img DOM-element, define its src attribute and we are good to go. Everything would happen automatically. However, there is a big security-issue with this approach. 

Take this hypothetical application for instance: We have an app that is used to manage pictures and this particular app can have multiple users. It makes sense that every user owns their own images, right? For privacy reasons user A could never consume the images of user B. Since images are loaded through the DOM with the img DOM-element, it might be hard to add authorization there. Take this angular snippet for instance:

```typescript
@Component({
    template: `
        <img [src]="img.src"/>
    `
})
export class FooComponent {
    img = {
        src: 'https://angular.io/assets/images/logos/angular/logo-nav@2x.png'
    }
}
```

Since the browser just loads this image through a normal HTTP call, how is the server supposed to know if user A or user B is trying to consume this image? The server needs some kind of authorization to be passed to it.

## Passing the authorization with Session cookies

We could use session cookies, which kinda works like this: The user authenticates with the backend of our app, receives a session cookie which will be passed with every future request. That way the backend knows who is trying to consume the resource in question.

Some could argue that session cookies are not to way the go, and stateless backends are better and more scalable.
If we use JWT or any other modern authentication system we would rather send **authorization-headers** instead of using the session cookie approach. The value that our server should receive in the authorization-header is called a **Bearer token**.

## Passing the authorization without cookies

Let's explore 2 different methods to pass Bearer tokens to the backend:

### Passing the token in the url

In this case we would just pass the token as a **query parameter** in the url. The previous snippet would now look like this:

```typescript
@Component({
    template: `
        <img [src]="img.src + '?bearer=' + bearToken"/>
    `
})
export class FooComponent {
    img = {
        src: 'https://angular.io/assets/images/logos/angular/logo-nav@2x.png'
    }
    bearerToken  = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
}
```

The url of the server request would now look like this: *https://angular.io/assets/images/logos/angular/logo-nav@2x.png?bearer=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...* 

Based on the token, the server knows which user tries to load the image and can block unauthorized users of fetching that image.

There are a few tradeoffs:

- The Token is there for everyone to see in the URL (less secure, but then again, they could still check it in the offline-storage)
- Every time the token changes, the cache would be cleared, since the url of the resource changes
- The backend would need a way to handle all the image calls that contain a bearer token in their url
- We need to load and parse the token everywhere we load images in our components
- Doesn't work for css, unless it's dynamically created

###  Handling the images with AJAX calls

In this approach we will create a generic way of handling image-security by using:

- A generic component
- AJAX calls with a blob responseType
- HTTP interceptors (introduced by the new HttpClient in angular 4.3)
- Data urls

We would like to consume images like this:

```typescript
@Component({
    template: `
          <secured-image [src]="img.src"></secured-image>
    `
})
export class FooComponent {
    img = {
        src: 'https://angular.io/assets/images/logos/angular/logo-nav@2x.png'
    }
}
```

Let's create a **secured-image** component as shown above.
This component needs to handle the following:

- Load an image through an AJAX call
- Create a data url based on a blob
- When the source changes:
    - Cancel the previous AJAX call (if it's still busy)
    - Start loading the new resource through AJAX
- When the component gets destroyed, cancel the current AJAX call (if it's still busy)

In the beginning, this might look like this:

```typescript
@Component({
  selector: 'secured-image',
  template: `
    <img [src]="dataUrl$|async"/>
  `
})
export class SecuredImageComponent implements OnChanges  {
  // This code block just creates an rxjs stream from the src
  // this makes sure that we can handle source changes
  // or even when the component gets destroyed
  // So basically turn src into src$
  @Input() private src: string;
  private src$ = new BehaviorSubject(this.src);
  ngOnChanges(): void {
    this.src$.next(this.src);
  }

  // this stream will contain the actual url that our img tag will load
  // everytime the src changes, the previous call would be canceled and the
  // new resource would be loaded
  dataUrl$ = this.src$.switchMap(url => this.loadImage(url))
  
  // we need HttpClient to load the image
  constructor(private httpClient: HttpClient) {
  }

  private loadImage(url: string): Observable<any> {
    return this.httpClient
      // load the image as a blob
      .get(url, {responseType: 'blob'})
      // create an object url of that blob that we can use in the src attribute
      .map(e => URL.createObjectURL(e))
  }
}

```

This pretty much covers everything, but if we check it in the browser we get the following error.
**WARNING: sanitizing unsafe URL value blob:https://localhost:4200/da89c71e-5df2-4842-af06-993cd5263471 (see http://g.co/ng/security#xss)**

Loading the image through AJAX does not work yet because we haven't sanitized the url yet. For that we need the **DomSanitizer** that angular provides us. This is a security mechanism to protect the app from XSS-attacks. We basically have to tell angular which urls to trust.

```typescript
export class SecuredImageComponent implements OnChanges  {
  ...
  // inject the domSanitizer here as well
  constructor(private httpClient: HttpClient, private domSanitizer: DomSanitizer) {
  }

  private loadImage(url: string): Observable<any> {
    return this.httpClient
      .get(...)
      // pass the url through the domSanitizer so angular knows he can parse it
      .map(e => this.domSanitizer.bypassSecurityTrustUrl(URL.createObjectURL(e)))
  }
}

```

We now have a fully working way of loading images through AJAX calls. However, we still haven't passed our Bearer token.
We could add the the authorization header in the get call directly, but let's find a cleaner solution.
Since we use httpClient, this opens up a few doors for us.
The new HttpClient, which was introduced in angular 4.3 offers a few new features. One of these features are **interceptors**.
Interceptors are a way to hook into http calls that are being made by the new HttpClient.

This would be a perfect solution to pass the bearer token wouldn't you agree?

Let's create the interceptor and register it to angular.

```typescript
// my-http.interceptor.ts
@Injectable()
export class MyHttpInterceptor implements HttpInterceptor {
  // intercept any http call done by the httpClient
  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    // fetch the bearer token from wherever you have stored it
    // NOTE: fetching it directly from window is not a good idea (demo purpose)
    const jwtToken = window.localStorage.getItem('jwtToken');

    // if there is a token, clone the request and set the correct
    // authorization header, if not => just use the old request
    const requestToHandle = jwtToken
      ? request.clone({
        headers: request.headers.set('authorization', `Bearer ${jwtToken}`)
      })
      : request;
    return next.handle(requestToHandle);
  }
}

// app.module.ts
@NgModule({
  ...
  // don't forget to import the HttpClientModule
  imports: [ BrowserModule, FormsModule, HttpClientModule ],
  providers: [{
    // register the interceptor to our angular module
    provide: HTTP_INTERCEPTORS, useClass: MyHttpInterceptor, multi: true
  }]
})
export class AppModule { }
```

If you would like to learn more about interceptors, check this [awesome article](https://juristr.com/blog/2017/08/intercept-http-requests-in-angular/) by [Juri Strumpflohner](https://twitter.com/juristr).
He also has an amazing egghead course on that subject.

Right now, every call that the secured-image component has initiated will load the image through AJAX with the right authorization header. That way the server can check who has been asking for that specific resource.

However, this approach also introduces a few tradeoffs:

- CORS headers for CDN resources. Since we are using a GET AJAX call there will be extra OPTIONS calls.

Extra advantages:
- It's easier to handle the loading of the image, show a spinner or animation
- Doesn't work for css, unless it's dynamically created

You can find the complete source of a working version in this [stackblitz example](https://stackblitz.com/edit/secure-image-loads).

## Thanks for reading
I hope you all liked this article, if you have questions, shoot!