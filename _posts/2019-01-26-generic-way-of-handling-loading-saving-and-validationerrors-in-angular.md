---
layout: post
title: A generic way of handling loading-status, saving-status and validation errors in Angular
published: true
description: In this article we are going to implement a generic solution on how to fix 3 common usecases that involve redundancy in CRUD applications.
author: brechtbilliet
comments: true
date: 2019-01-25
subclass: 'post'
categories: 'angular'
disqus: true
tags: Angular Architecture errorhandling interceptors decorators
cover: 'assets/images/cover/cover5.jpg'
---

When writing Angular applications there are always pieces of functionality that are being rewritten over and over again. 3 common usecases are:

- Showing a *loading* status
- Showing an *acting* status (whether the user is adding, updating or removing data)
- Showing validation errors

In this article we are going to implement a generic solution on how to fix these usecases.

## A non generic way of loading, saving and handling validation errors in Angular

Before we jump to the solution, let's have a look at the impact of a non generic way of handling the previous called functionalities.
A solution that is often used might look like the following: (keep in mind that this functionality has to be implemented over and over again for every component) 

```typescript
ngOnInit(): void {
    this.loading = true;
    const usersCompleted = false;
    const citiesCompleted = false;
    this.userService.fetch().subscribe(users => {
        usersCompleted = true;
        this.loading = usersCompleted && citiesCompleted;
    });
    this.citiesService.fetch().subscribe(cities => {
        citiesCompleted = true;
        this.loading = usersCompleted && citiesCompleted;
    });
}
```
We have to keep track of which call completes first because we can't set loading to false, if there is call still busy. What happens when there is an error? We would have to implement that as well.

Now this code is only for fetching two lists of data, this becomes ugly pretty quickly and the worst thing about this is that we have to reimplement that for every component that does data fetching.

It becomes even worse if we want to update, add and remove data. Imagine that we have to handle validation errors as well:

```typescript
remove(user: User): void {
    // TODO: set acting to true
    this.userService.remove(user).subscribe(res => {
        // TODO: set acting to false
    });
}
update(user: User): void {
    // TODO: set acting to true
    this.userService.update(user).subscribe(res => {
        // TODO: set acting to false
        // TODO: handle validation errors
    });
}
add(user: User): void {
    // TODO: set acting to true
    this.userService.add(user).subscribe(res => {
        // TODO: set acting to false
        // TODO: handle validation errors
    });
}
```

To handle validation errors we have to check if the HTTP status code is 400, manually map the data etc.

These code samples are in this article to prove a point. **It's dirty redundant logic that we have to implement over and over again**

## Let's clean this up

To achieve this we will use an Angular *service* in combination with an angular *interceptor* and Typescript *decorators*.

The first thing we need is a `HttpStatusService` that exposes 3 observables:

- `loading$`: whether the user is fetching data
- `acting$`: whether the user is removing, updating or adding data
- `validationErrors$`: whether there are validation errors or not

```typescript
// http-status.service.ts
@Injectable({
  // important to provide this service to the 
  // injector of the root module 
  providedIn: 'root'
})
export class HttpStatusService {
  // regular subject because we don't want to replay
  // the validationerrors
  private validationErrorsSub$ 
    = new Subject<ValidationError[]>();

  // 2 subjects that replays the last value 
  // (ideal for state)
  private loadingSub$ = new ReplaySubject<boolean>(1);
  private actingSub$ = new ReplaySubject<boolean>(1);

  // we don't want to expose the subject for
  // encapsulation purposes. That's why we convert them
  // into observables
  getvalidationErrors$ = 
    this.validationErrorsSub$.asObservable();
  loading$ = 
    this.loadingSub$.pipe(distinctUntilChanged());
  acting$ = 
    this.actingSub$.pipe(distinctUntilChanged());

  // these are just some regular setters to next 
  // the values in our subjects
  set validationErrors(errors: ValidationError[]) {
    this.validationErrorsSub$.next(errors);
  }

  set loading(val: boolean) {
    this.loadingSub$.next(val);
  }

  set acting(val: boolean) {
    this.actingSub$.next(val);
  }
}
```

So we have a service that basically holds the state of our three statuses.
Now we still have to make sure that the setters of these observables are being called at the right place and the right time.
We don't want to manually implement that for every call, so let's create an interceptor for that.

```typescript
@Injectable({
    providedIn: 'root'
})
export class HttpStatusInterceptor implements HttpInterceptor {
  // keep track of the loading calls
  private loadingCalls = 0; 
  // keep track of the acting calls
  private actingCalls = 0; 

  constructor(
    private httpStatusService: HttpStatusService
  ) {}

  private changeStatus(val: boolean, method: string): void {
    if (['POST', 'PUT', 'DELETE', 'PATCH']
      .indexOf(method) > -1) {
      val ? this.actingCalls++ : this.actingCalls--;
      this.httpStatusService.acting = this.actingCalls > 0;
    } else if (method === 'GET') {
      val ? this.loadingCalls++ : this.loadingCalls--;
      this.httpStatusService.loading = this.loadingCalls > 0;
    }
  }
  ...
}

```

As we can see, we have created a private `changeStatus()` function that will use the `loading` and `acting` 
setters of our `HttpStatusInterceptor` class.

If the HTTP-method is `POST`, `PUT`, `DELETE` or `PATCH` we have to update the counter of the `actingCalls` and if that count is bigger then 0, it means the user is acting and we have to update the `acting` property of the `HttpStatusService`.

If the HTTP-method is `GET` it should do the same for the `loadingCalls` property and `loading` setter of the `HttpStatusService`.

Of course we still have to implement the `intercept` function. Every time we intercept a call we have to change a status to `true`.
Because this means that the user is loading or acting.
The goal of an interceptor is to intercept a request, clone that request, do something with it and return it.
The `handle` function of the `HttpHandler` returns an observable. This is the perfect place to apply a `finalize` operator,
which we can use to set a status to `false`.

```typescript
...
intercept(
  req: HttpRequest<any>,
  next: HttpHandler
): Observable<HttpEvent<any>> {
  // there is a new request, so we are definitely
  // loading or acting, we have to change the status
  this.changeStatus(true, req.method);
  return next.handle(req.clone()).pipe(
    // when the request completes, errors or times out,
    // we have to change the status as well
    finalize(() => {
      this.changeStatus(false, req.method);
    })
  );
}

```

This should automatically update the `loading$` and `acting$` observables in the `HttpStatusService`.
However, we have not fixed our validation errors issue yet. For that we can use the `catchError` operator that we place before the finalize operator:

```typescript
intercept(
  req: HttpRequest<any>,
  next: HttpHandler
): Observable<HttpEvent<any>> {
  ..
  return next.handle(req.clone()).pipe(
    // catch the error
    catchError(e => {
      // if bad request > validation erors
      if (e.status === 400) { 
        // use the validationErrors setter to update
        this.httpStatusService.validationErrors = 
          e.error.validationErrors;
        // make sure that this result never 
        // reaches the component
        return NEVER;
      }
      // throw the error back
      // or put that in the `HttpStatusService` as well ;-)      
      return throwError(e); 
    }),
    finalize(...)
  );
}
```

The entire interceptor class looks like this:

```typescript
@Injectable({
    providedIn: 'root'
})
export class HttpStatusInterceptor implements HttpInterceptor {
  private loadingCalls = 0; 
  private actingCalls = 0; 

  constructor(
    private httpStatusService: HttpStatusService
  ) {}

  private changeStatus(v: boolean, method: string): void {
    if (['POST', 'PUT', 'DELETE', 'PATCH']
      .indexOf(method) > -1) {
      v ? this.actingCalls++ : this.actingCalls--;
      this.httpStatusService.acting = this.actingCalls > 0;
    } else if (method === 'GET') {
      v ? this.loadingCalls++ : this.loadingCalls--;
      this.httpStatusService.loading = this.loadingCalls > 0;
    }
  }

  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    return next.handle(req.clone()).pipe(
      catchError(e => {
        if (e.status === 400) { 
          this.httpStatusService.validationErrors = 
            e.error.validationErrors;
          return NEVER;
        }
        return throwError(e); 
      }),
      finalize(() => {
        this.changeStatus(false, req.method);
      })
    );
  }
}
```

When registering the interceptor to the root module, the observables in the `HttpStatusService` will be updated automatically.

So in the rootModule we have to add the following code to the `providers` property of the `@NgModule` decorator:

```typescript
providers: [
  ...
  {
    provide: HTTP_INTERCEPTORS,
    multi: true,
    deps: [HttpStatusService],
    useClass: HttpStatusInterceptor
  }
]
```

We can now use the `HttpStatusService` in our code as easy as this:

```typescript
export class UserComponent {
  loading$ = this.httpStatus.loading;
  validationErrors$ = this.httpStatus.validationErrors;
  acting$ = this.httpStatus.loading;

  constructor(
    private httpStatusService: HttpStatusService) {
  }
}
```

We now have 3 observables that can easily be consumed in the template of the component with the use of the [async](https://angular.io/api/common/AsyncPipe) pipe. Here is an example:

```html
<my-spinner *ngIf="loading$ | async"></my-spinner>
<my-user-form 
    [validationErrors]="validationErrors$ | async"
    [disabled]="acting$ | async"></my-user-form>

```

## Optimizing with decorators

We have cleaned up a lot, we found an easy way to get the httpstatuses to the component, but we can make it even simpler with the use of decorators. let's refactor the `UserComponent` class accordingly:

```typescript
export class UserComponent {
  @Loading()loading$;
  @ValidationErrors() validationErrors$;
  @Acting() acting$;
}
```

This is very declarative way of working. The component is way cleaner and we don't have to inject the `HttpStatusService` anymore.

But how do we create these decorators? I'm glad you asked, it's pretty easy. A property decorator is simply a function that returns a function that gets the target and key as arguments.

```typescript
export function Loading() {
  return function (target: any, key: string): void {
    // in this case the target is the component
    // instance and key the property name
    // target: userComponent, key: loading$
    // now we have to set the property to the actual 
    // loading$ observable that lives in the HttpStatusService
    target[key] = // todo
  }
};
```

To set the property of the target we need to inject the `HttpStatusService` instance that is registered on the root injector of our application. After all that's the instance that contains the actual state. Currently there is no easy way to do that.
Until Angular provides us with functionality like [that](https://github.com/angular/angular/issues/23301) we can use the following solution:

Next to the `http-status.service` file, create a file called `root-injector.ts` and add the following code:

```typescript
import { Injector } from '@angular/core';

export let rootInjector: Injector;
export function setRootInjector(injector: Injector): void {
  rootInjector = injector;
}
```

The `setRootInjector()` function will be used by the rootModule to set the `rootInjector` variable that we expose here.
To make it work the rootModule will have to call the `setRootInjector()` function like this:

```typescript
@NgModule({
    ...
})
export class AppModule {
  constructor(private injector: Injector) {
    setRootInjector(injector);
  }
}

```

The last step is to actually use the `rootInjector` variable inside the decorators. The result looks like this:


```typescript
// loading.decorator.ts
export function Loading() {
  return function (target: any, key: string): void {
    const service = rootInjector.get(HttpStatusService);
    target[key] = service.loading$;
  }
};

// acting.decorator.ts
export function Acting() {
  return function (target: any, key: string): void {
    const service = rootInjector.get(HttpStatusService);
    target[key] = service.acting$;
  }
};

// validation-errors.decorator.ts
export function ValidationErrors() {
  return function (target: any, key: string): void {
    const service = rootInjector.get(HttpStatusService);
    target[key] = service.validationErrors$;
  }
};
```

## Conclusion

We have learned that we can remove the redundancy that comes with loading, acting and error handling statuses almost completely by the use of an interceptor, a simple service and a few decorators.

I hope you liked it!

## Special thanks

A very special thanks to the reviewers:

- [Vitallii Bobrov (@bobrov1989)](https://twitter.com/bobrov1989)
- [David Müllerchen (@webdave_de)](https://twitter.com/webdave_de)
- [Maarten Tibau (@maartentibau)](https://twitter.com/maartentibau)
- [Dominic Elm (@elmd_)](https://twitter.com/elmd_)
- [Fabian Gosebrink (@FabianGosebrink)](https://twitter.com/fabiangosebrink)
