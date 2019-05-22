---
layout: post
title: The @Injectable decorator and its relationship to dependency injection
published: true
description:  A post on the relationship between the @Injectable decorator and dependency injection
author: kwintenp
comments: true
date: 2019-05-19
subclass: 'post'
categories: 'RxJS'
disqus: true
tags: RxJS
cover: 'assets/images/cover/cover5.jpg'
---

Last week, I had the pleasure of being a jury member of the Angular Challenge 2019. This is a competition organized by the organizers of NG-BE (<a href="https://twitter.com/jvandemo" target="_blank">Jurgen</a> en <a href="https://twitter.com/samvloeberghs" target="_blank">Sam</a>) where Angular developers can come together to test their knowledge. The first step is online and the best participants are asked to join in for an in-person contest.

As a jury member, it was my duty to create questions for both these rounds. One of my questions that was used last week seemed to have caused a little controversy. In this post, I would like to explain the question, the correct answers and apologize for those who found the question to be unclear or misleading. 

## The question 

Let's look at the **question**:

Which of the following statements regarding the `@Injectable` decorator
are true?

<strong>Answers:</strong>

1. @Injectable needs to be added to every service.
2. @Injectable needs to be added to services that use DI.
3. @Injectable is optional if you don't use the 'providedIn' option.
4. @Injectable in combination with the 'providedIn' option means the
service doesn't need to be added in the providers' array of a
module.


As you can see from the question (and title of this post), it was about the `@Injectable` decorator and its effect on dependency injection(DI) in Angular. The correct answer to this question is option 2 and 4. 

The reason that some people answered this wrong is that they misunderstood the relation between what the `@Injectable` operator really does and DI in Angular. So let's dive in a little deeper here. 

## Adding @Injectable is not registering the service

In any IOC (inversion of control) container (used for DI), you always need two things. First of all, you need a token. A token is something unique that is used to register something with the container. Next thing we need is a provider that explains to the DI container how it can create an instance of a certain dependency. 

In Angular, we can register a service for a token and pass it a provider in two different ways. 

We can register a service with a specific `@NgModule` by passing a service to the providers' array. In this scenario, our token is the typescript type `SomeService`. The provider is the `useClass` strategy which also points to the `SomeService`. This provider strategy basically says that this dependency can be instantiated by `new ... ()`ing it.

```typescript
@NgModule({
  ...
  providers: [
    // long hand syntax
  	{provide: SomeService, useClass: SomeService},
  	// short hand syntax
  	SomeService
  ],
})
```
Because the token and the provider are the same, we can use the short-hand syntax.

**Note:** The `@NgModule` is not the only place where you can add a service to the providers array. The concept of registering is the same with `@Components` and `@Directives`.

The second way that we can register a service is by using the `providedIn` option.

```typescript
@Injectable({
  providedIn: 'root'
})
export class SomeService {

  constructor() { }
}
```

In this scenario, we are registering the service with the root `@NgModule` (but only if the service is injected in a service, component or directive that Angular knows). The token here is `SomeService` and the same strategy as in the example above will be used. This is called a tree-shakable provider.

### But what about @Injectable?

As you can see, I've not yet mentioned what the `@Injectable` decorator does (without the `providedIn` at least). That's because the `@Injectable` decorator has nothing to do with registering services with the IOC container. So what is it used for?

Well, it is pretty standard that our `SomeService` can also have some dependencies. For instance, we can use the `HttpClient`.

```typescript
import { HttpClient } from '@angular/common/http';

@Injectable()
export class SomeService {

  constructor(private httpClient: HttpClient) { }
}
```

Let's try and reason about what happens when we want to get an instance of this `SomeService`. First of all, this `SomeService` will need to be registered with Angular so let's assume it is. When Angular wants to create this service, it will need to pass it an instance of the `HttpClient`. So how can it do that?

Well, first of all, Angular will need to know which dependency is requested. Remember that we use a token to register a dependency? Well, with the same token (and this is really important), we can ask Angular for an instance of that dependency. So what Angular can do is, by investigating our constructor, it can see that we are requesting a service for the token `HttpClient`. And if a service has been registered with that token, Angular will give use it to instantiate the `SomeService` and give us the instance it created.

But wait a minute 🤔, the token we are talking about is a Typescript type. And when our code is running in the browser, we are running Javascript. And when we transpile from Typescript to Javascript, we lose all the types. So how can Angular know what token we are requesting in our constructor???

Thanks to the `@Injectable` decorator! The decorator is going to add metadata information to our transpiled class to make sure that the type information that is needed to identify the tokens is not lost. To prove this, let's take a look at the transpiled code for our `SomeService`.

```typescript
var SomeService = /** @class */ (function () {
    function SomeService(httpClient) {
        this.httpClient = httpClient;
    }
    SomeService = __decorate([
        Object(_angular_core__WEBPACK_IMPORTED_MODULE_0__["Injectable"])(),
        __metadata(
        	"design:paramtypes", 
        	[_angular_common_http__WEBPACK_IMPORTED_MODULE_1__["HttpClient"]]
        )
    ], SomeService);
    return SomeService;
}());
```

As you can see, the decorator adds metadata so that when this service is instantiated, Angular knows what services are requested and we don't lose the token information. 

Important to note here is that Angular is NOT using the `@Injectable` decorator to register the service. It is only using the decorator so it knows what dependencies the service actually requests. 

**Note:** Other decorators such as `@Component` will also add this metadata to the transpiled class (as components can also have dependencies of course).

### Back to the question

Now, with all of this in mind, let's look back at the **question**.

Which of the following statements regarding the `@Injectable` decorator
are true?

**Answers:**

1. @Injectable needs to be added to every service.
2. @Injectable needs to be added to services that use DI.
3. @Injectable is optional if you don't use the 'providedIn' option.
4. @Injectable in combination with the 'providedIn' option means the
service doesn't need to be added in the providers array of a
module.

The first and second option are basically the opposite, meaning if one is true the other is false and vice versa. If we have a service, that is registered with Angular and doesn't require DI (meaning it doesn't have any dependencies itself), we don't need the `@Injectable` decorator. This decorator only gives Angular information about the dependencies this service wants, but if it doesn't have any, then the decorator is optional! So the second option is correct and the first one false.

The third option is false because if our service has dependencies, that means the `@Injectable` operator is needed! Otherwise Angular doesn't know the tokens that are needed. This is unrelated to whether or not the `providedIn` option is used.

The fourth option is correct as we've seen above.

To demonstrate all of this, you can use this <a href="https://stackblitz.com/edit/angular-pr34ij" target="_blank">stackblitz</a>.

## Conclusion

The `@Injectable` decorator is related to but not necessarily crucial for every service when using DI in Angular. I must admit that, in hindsight, the question is a little misleading in the sense that for option 2 stating 'services that use DI' is not the best way to phrase it. I meant a service that has dependencies of its own. And secondly, it's always best to add the decorator anyway. There isn't really a benefit in omitting it (bytes gained is marginal). If you wouldn't have it and you add a dependency later on, you might get some weird bugs as well.

So to the participants that answered this question wrong because of the question being unclear, I do apologize! 

**Note:** The Angular docs actually make some wrong statements about the `@Injectable` decorator. Frederik Prijck, who participated in the challenge is going to create a PR to make sure it gets fixed ASAP!

I hope this clarifies it a bit. And even though you probably best add the decorator to every service, understanding how it works under the hood is in my opinion still a crucial aspect to become a better Angular developer!






