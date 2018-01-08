---
layout: post
cover: false
title: How share() can reduce network requests
date:   2016-12-20
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS Angular
cover: 'assets/images/cover/cover8.jpg'
---
As you hopefully all know, observables you get back from the Angular 2 Http service are cold. A cold observable only starts when you subscribe to to it and is unicast (for every subscription to the Http observable, a network call is triggered).

**Note:** If you want to dive deeper into hot vs cold observables, Christoph Burgdorf from Thoughtram wrote <a href="http://blog.thoughtram.io/angular/2016/06/16/cold-vs-hot-observables.html" target="_blank">an awesome article </a> on the subject (Rx pun intended :)).

#### Project setup

To demonstrate how the `share()` operator can reduce our number of network calls, I've created a demo application (code available on <a href="https://github.com/KwintenP/save-netwerk-requests-with-share" target="_blank">Github</a>). The setup looks like this:

![setup](https://www.dropbox.com/s/d87po1zbp2ri11u/Screenshot%202016-12-15%2020.08.33.png?raw=1)

As you can see, we have an app component, which is smart, and three dumb components. During startup, the app component will randomly fetch one of the Star Wars characters by calling the StarWarsService.

```typescript
// app.component.ts
this.character$ =
     this.starWarsService.getCharacter(this.generateNumber());
```

The StarWarsService uses Angular's Http and returns a character from the series using the swapi.co API.

```typescript
// star-wars.service.ts
public getCharacter(id: number): Observable< StarWarsCharacter > {
    return this.http.get('https://swapi.co/api/people/' + id)
      .map((response: Response) => response.json());
}
```


The app component creates three new observables by mapping the `character$` source observable.

```typescript
// app.component.ts
this.name$ = this.character$.map(character => character.name);
this.birthDate$ = this.character$.map(character => character.birth_year);
this.gender$ = this.character$.map(character => character.gender);
```

The data is passed to the dumb components using the `async` pipe.

```html
<app-character-name [name]="name$ | async">
</app-character-name>
<app-character-birthdate [birthDate]="birthDate$ | async">
</app-character-birthdate>

<button type="button" (click)="toggleEnabled()">
     Enable gender component
</button>
<app-character-gender *ngIf="enabled" [gender]="gender$ | async">
</app-character-gender>
```

As you can see in the snippet above, the gender component is only rendered **after** the button has been clicked. This way we can simulate a 'delayed subscription'. We'll see why this is important in a second.

#### Potential pitfall
If we run this code, everything works perfectly. But if we look at the network tab, we can see some strange behaviour.

![setup](https://www.dropbox.com/s/j5etmgqz668v4wp/Dec-15-2016%2020-00-10.gif?raw=1)

When the page is refreshed, we already see two calls to the backend. If we click the button to enable the gender component, we see another request being send. Why do we see multiple backend requests if we're only asking for one `character$`?

The reason is simple. The `character$` is a **cold observable**. If you subscribe to a cold http observable twice, you'll have two backend requests.
We map this `character$` or source observable using the `map` operator to three different observables. Every subscription to one of these three observables (which is done by the async pipe), is basically the same as subscribing to the `character$` observable directly. And because the `character$` observable is a cold observable, you see multiple requests.
The problem with this is that the underlying subscription to the source observable is **not shared**.

#### How can we fix this (kind of)

Let's change the observable that the StarWarsService returns like this:

```typescript
return this.http.get('https://swapi.co/api/people/' + id)
      .map((response: Response) => response.json())
      .share();
```

We added the `share()` operator. This is an alias for doing `publish().refCount()`. This will make the `character$` a hot observable that starts emitting events as soon as the first one subscribes. The `character$` observable will be a 'shared' one. Let's first see what this means and explain afterwards:

![setup](https://www.dropbox.com/s/o3i03kudmutt1gk/Dec-15-2016%2019-57-47.gif?raw=1)

Now we can see that, on the initial page load, there is only one request. That's because the `share()` operator will share the underlying subscription between its listeners. As soon as the first one subscribes, it will listen to the Http observable and thus trigger one backend request. Everybody else who starts listening afterwards, will get the same result. Hence we only see a single request to the swapi.co API.


But, when I click on the button to enable the gender component, we can see that a second request is triggered. This means our `share()` operator doesn't help us entirely. It turns out that with `share()`, if the source observable completes, the underlying subscription is gone as well. So when the gender component gets rendered, the original request is long finished and completed. For every new subscription after this completion, in our case via the `gender$` observable, the `share()` operator will start a new backend request.

This can perfectly be expected behaviour, if you want it. In some cases though, you don't.

#### publishReplay to the rescue

Let's change the observable from the StarWarsService one more time:

```typescript
let obs$: ConnectableObservable<StarWarsCharacter> =
     this.http.get('https://swapi.co/api/people/' + id)
         .map((response: Response) => response.json())
         .publishReplay();
obs$.connect();
return obs$;
```

I removed the `share()` operator and used `publishReplay()` instead. This will return an observable that will subscribe to the source observable as soon as you `connect()` it. In our case, this needs to happen immediately, so I call it before returning.
Let's see what this does:

![setup](https://www.dropbox.com/s/5j4381vj8439dbb/Dec-15-2016%2019-55-20.gif?raw=1)

Alright! Now we only have one request at startup time and we do not have a new one when we render the gender component. The `publishReplay()` operator doesn't care if the Http observable completes or not. For every new subscription, it will just return the previous result.

#### Conclusion

* If your source observable 'splits' into multiple observables, using the `share()` operator will reduce network requests.
* If you have subscriptions which are delayed due to delayed rendering, you might want to use the `publishReplay()` operator to avoid extra requests.

**NOTE:** Thanks to <a href="https://twitter.com/juristr">@juristr</a> for the review!







