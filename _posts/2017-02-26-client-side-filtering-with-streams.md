---
layout: post
cover: false
title: Client side filtering using streams
date:   2017-02-15
subclass: 'post'
categories: 'casper'
published: true
disqus: true
navigation: True
logo: 'assets/images/strongbrewlogo.png'
author: kwintenp
tags: RxJS
cover: 'assets/images/cover/cover8.jpg'
---

I have been coaching people in using RxJS for a while now. During this time, I've noticed that the hardest part for people to learn is not the API, concept or operators but the paradigm switch. Thinking reactively is not something that comes easily and you really have to get your hands dirty to get there. 

Below I will show a piece of code that is used to do some basic client side filtering. It's a snippet of code from somebody that I was coaching which perfecly shows how someone just starting with RxJS often handles this situation. Later on, we will update this to show how you would implement this with the reactive paradigm.

### Example case

The example application we will be using is really easy. We have service which fetches a number of Star Wars characters from the swapi.com API. We will show these in a list and provide a select element to filter the fetched characters based on the gender.
The screen looks like this:

![example-app](https://www.dropbox.com/s/2s9e877rpdaa5w0/Screenshot%202017-02-25%2011.16.57.png?raw=1)

You can find the live example <a href="http://blog-kwintenp-examples.surge.sh/client-side-filter/withStream" target="_blank">here</a>.

### Client side filtering without streams

First we are going to look at an example where we implement the client side filtering without streams. Of course, we are going to use a stream to fetch the data from the backend, but afterwards, the implementation will be imperative.

```typescript
// keep a local list of all the characters
characters: Array<StarWarsCharacter>;
// keep a list of all the filtered characters
filteredCharacters: Array<StarWarsCharacter>;

constructor(private starWarsService: StarWarsService) {}

// at startup time, we fetch the characters and save them
// to our local copy. We keep a local copy of the entire
// array since we will need it later on when the filter changes
ngOnInit() {
  this.starWarsService.getCharacters()
    .subscribe((fetchedCharacters) => {
        this.characters = fetchedCharacters;
        this.filteredCharacters = fetchedCharacters;
    })
}

// when the filter value changes, we filter the local list of
// characters and save the result to the
// filteredCharacters array. Here we reuse the entire array
// to create a new one.
filterChanged(value: string)
  if (value === "All") {
    this.filteredCharacters = this.characters;
  } else {
    this.filteredCharacters =
       this.characters.filter(
            (character: StarWarsCharacter) => {
               character.gender.toLowerCase() === value.toLowerCase()
            }
       );
  }
}
```
At startup time, we fetch the characters and save them in two local arrays, `characters` and `filteredCharacters`. When the filter actually changes, we use the local copy of the characters to filter out all the correct ones and create a new array. We then assign this new array to the `filteredCharacters` reference.
The component's html looks like this:

```html
<h1>Client side filtering without streams example</h1>
<div class="row">
  <div class="col-sm-3">
    <!-- Component that holds the select and throws an event -->
    <!-- when it changes -->
    <app-gender-filter (filterChange)="filterChanged($event)">
    </app-gender-filter>
  </div>
  <div class="col-sm-9"></div>
  <div class="col-sm-6">
    <!-- component that holds the list and displays the -->
    <!-- characters -->
    <app-character-list [characters]="filteredCharacters">
    </app-character-list>
  </div>
</div>

```

While this all works perfectly, it's not the best solution possible with the reactive paradigm in mind. We have to hold a local copy of the characters array, which kind of bugs me.
It's also not really flexible. Here, we are fetching the characters via a backend call. This will thus only hold one result. But what if it's an observable we get from Firebase? In that case the characters array can change as well. To be able to update the view properly when the characters change, we would also have to keep a local copy of the filter to update the `filteredCharacters` array reference accordingly.
And what if you would have a multitude of filters... This would result in a multitude of local variables to keep track of.

Using streams we can make this much more flexible!

### Client side filtering with streams

Let's first of all try to think what should happen by thinking in streams of data. We will then try to reason about how we can use these streams, combine them and create a result.

If you think about it, we have two inputs that might change our view. On the one hand, we have our list of characters, which should be displayed. And on the other hand we have the dropdown which might filter this list. 

Let's look at the marble diagrams of what these streams might look like:

![marble-diagram](https://www.dropbox.com/s/89blsfj9aoybdg0/Screenshot%202017-03-04%2016.10.54.png?raw=1)

The resulting stream we want is one that holds an array with the filtered characters. If you think about the two input streams we just created, this resulting stream is actually a combination of both of the input streams.
If we have characters, we want to combine these with the filter and display them onto the screen. If the filter changes, we want to re-execute this logic. If we would have new characters (think about a stream from Firebase), we would also want to re-execute this logic.

It turns out that RxJS provides us with a perfect operator to do something like this: `combineLatest`. This operator merges streams by executing a projector function on the latest values of these streams.
The resulting stream looks like this:

![marble-diagram](https://www.dropbox.com/s/zhj0xvz6d5e84m4/Screenshot%202017-03-04%2016.12.24.png?raw=1)

**Note:** You can notice here that the result stream only holds a value when both of the streams have emitted a single value. This is a precondition of the `combineLatest` operator. It will only emit an event onto the newly created stream if all source observables have emitted at least one element. If we think about our example, this means that the gender filter should already have a value to start with.

Let's take a look at the code!

```typescript
export class ClientSideFilterComponent implements OnInit {
  filter$: BehaviorSubject<string>;
  characters$: Observable<StarWarsCharacter[]>;
  filteredCharacters$: Observable<StarWarsCharacter[]>;

  constructor(private starWarsService: StarWarsService) {
  }

  ngOnInit() {
    // We create a stream ourselves to map an event form the child
    // component to a stream of 'filter values'.
    // We use a BehaviorSubject because this will have an initial 
    // value. This is important because the combineLatest operator
    // we will use below only works if every stream has emitted 
    // at least one value.
    this.filter$ = new BehaviorSubject("All");

    // we keep the stream containing our characters
    this.characters$  = this.starWarsService.getCharacters();

    // we create a new stream based on the two input
    // streams we defined
    this.filteredCharacters$ = this.createFilterCharacters(
            this.filter$,
            this.characters$
    );
  }

  public createFilterCharacters(
            filter$: Observable<string>,
            characters$: Observable<StarWarsCharacter[]>) {
    // We combine both of the input streams using the combineLatest
    // operator. Every time one of the two streams we are combining
    // here changes value, the project function is re-executed and
    // the result stream will get a new value. In our case this is
    // a new array with all the filtered characters.
    return characters$.combineLatest(
      filter$, (characters: StarWarsCharacter[], filter: string) => {
        // this is the project function where we imperatively
        // implement the filtering logic
        if (filter === "All") return characters;
        return characters.filter(
                (character: StarWarsCharacter) =>
                    character.gender.toLowerCase()
                        === filter.toLowerCase()
        );
      });
  }

  filterChanged(value: string) {
    // Everytime we have new value, we pass it to the filter$
    this.filter$.next(value);
  })
}
```
We can use the new stream we created to bind in the view using the async pipe from Angular like this:

```html
<h1>Client side filtering with streams example</h1>
<div class="row">
  <div class="col-sm-3">
    <app-gender-filter (filterChange)="filterChanged($event)">
    </app-gender-filter>
  </div>
  <div class="col-sm-9"></div>
  <div class="col-sm-6">
    <!-- Using the async pipe we bind it in the app-character-list -->
    <!-- component -->
    <app-character-list [characters]="filteredCharacters$ | async">
    </app-character-list>
  </div>
</div>

```

If we look at the code we have now, it's much more flexible. We do not have to write any extra code if the `characters$`  would have any new values. We do not need to hold any local copies (this is done implicitely by the `combineLatest` operator). If we would want to add new filters, it's just a matter of adding another stream to the `combineLatest$` operator.
We also don't have to manually unsubscribe from the filteredCharacters$, the async pipe handles this for us.

### Conclusion
By thinking in input streams and output streams, we were able to map the inputs we had to a result. We bound the stream in the view layer. Using streams up until the html of our component, we eliminated the need for local copies of data and made our code more robust and open to changes. If we now want to add another filter on top of the list, it's a piece of cake!

You can find the code on <a href="https://github.com/KwintenP/blog-examples/tree/master/src/app/home/client-side-filter" target="_blank"> Github </a>.
