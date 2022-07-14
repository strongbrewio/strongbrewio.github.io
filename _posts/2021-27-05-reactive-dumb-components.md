---
layout: post
title: Reactive dumb components
description:  How to make dumb components reactive without imperative ngOnChanges programming
published: true
author: brechtbilliet
comments: true
date: 2021-05-17
subclass: 'post'
categories: 'Rxjs'
disqus: true
tags: Rxjs
cover: 'assets/images/cover/cover4.jpg'
---

## Reactive dumb components

When we write RxJS logic inside our presentation layer we mostly wind up with reactive flows inside our smart components (container components). The reasoning behind this, is quite easy:

Because these smart components are the orchestration vessels of our application, they communicate with reactive asynchronous flows such as XHR-calls and state management, but also with events emitted by the dumb components.

That being said, dumb components may also benefit from reactive flows. Especially when the dumb component has multiple `@Input()` properties that rely on different sets of asynchronous data.
The usual way of handling these types of changes is the use of imperative `if else code` inside the `ngOnChanges` lifecycle hook.

Take this situation for example: We have a `CompanyDetail` component that has the responsibility to visualize a company with all its sites. 
The component has a previous and a next button to navigate between the sites, we also want to show how many sites there are and we want to show the name and address of every specific site.
The `sites` `@Input()` property is asynchronous because it is dependant on an XHR request and the `currentSiteId` `@Input()` property is asynchronous because it is dependant on the router params which is an observable behind the scenes and will change over time when the user is navigating between the different sites of a company.

For this component to properly work we have to make sure that the `sites` property has a value at the right time and the `currentSiteId` also has a value at the right time, we can already imagine racing conditions etc.
The component is also responsibile to calculate whether the previous button and next button are disabled or not. When these buttons are clicked, it also has to calculate the next or previous siteId to emit that value to its smart component.

Here we can see the app inside this [Stackblitz example](https://stackblitz.com/edit/reactive-dumb-components?file=src%2Fapp%2Fcontainers%2Fcompany%2Fcompany.component.ts)


This is the smart component that will use the `app-company-detail` as a dumb component.

```typescript
@Component({
  selector: 'app-company',
  template: `
    <app-company-detail
      [currentSiteId]="currentSiteId$ | async"
      [sites]="sites$ | async"
      (siteChanged)="siteChanged($event)"
    ></app-company-detail>
  `,
  styleUrls: ['./company.component.css']
})
export class CompanyComponent {
  // fetch the sites
  sites$ = this.sitesService.getSites();
  // get the asynchronous siteId from the router params
  currentSiteId$ = this.activatedRoute.params.pipe(map(p => p.siteId));

  // the dumb app-company-detail component is responsible
  // to calculate the siteId that we need to go to
  siteChanged(id: string): void {
    this.router.navigate([id]);
  }

  constructor(
    private sitesService: SitesService,
    private router: Router,
    private activatedRoute: ActivatedRoute
  ) {}
}
```

The imperative implementation of the dumb component looks like this:

```typescript
@Component({
  selector: 'app-company-detail',
  template: `
    <button [disabled]="previousDisabled" (click)="previousClicked()">
      Previous site
    </button>
    <button [disabled]="nextDisabled" (click)="nextClicked()">
      Next site
    </button>
    {{ currentSiteNumber }} / {{sites?.length}}
    <h2>{{currentSite?.name}}</h2>
    <p>Address: {{currentSite?.address}}</p>
  `,
  styleUrls: ['./company-detail.component.css']
})
export class CompanyDetailComponent implements OnChanges{
  @Input() currentSiteId: string;
  @Input() sites: any[];
  @Output() siteChanged = new EventEmitter<string>();

  // we need to keep track of 5 different local properties
  // and calculate and set their values at the right time
  currentIndex = 0;
  previousDisabled: boolean;
  nextDisabled: boolean;
  currentSite: any;
  currentSiteNumber: number;

  ngOnChanges(): void {
    // this can become complex really fast
    if(this.currentSiteId && this.sites?.length > 0){
      this.currentIndex = this.sites?.map(site => site?.id).indexOf(this.currentSiteId);    
      this.currentSite = this.sites[this.currentIndex];
      this.currentSiteNumber = this.currentIndex + 1;
      this.previousDisabled = this.currentIndex === 0;
      this.nextDisabled = this.currentIndex === this.sites?.length -1
    }
  }

  previousClicked(): void {
      this.siteChanged.emit(this.sites[this.currentIndex -1].id);    
  }

  nextClicked(): void {
      this.siteChanged.emit(this.sites[this.currentIndex +1].id);    
  }
}
```

We can see that all the calculation is happening inside the ngOnChanges.
Although this solution is perfectly fine, there is a more reactive way of approaching this. For this specific example it could be overkill but it might teach you how to tackle more complex situations when tackling `@Input()` changes. 

Think about a complex calendar view that has to do a bunch of calculations based on dozens of `@Input()` properties for instance, where a bunch of them are happening synchronous and there are async properties depending on other async properties.
Think about `@Input()` components that are useless unless other `@Input()` components have their values. Or what if you like to combine the `@Input()` properties with other async objects inside the dumb component. In the following example every piece of code inside this component is a stream. The first thing we want to do is create observables from the `@Input()` properties. I like to use this approach when multiple `@Input()` properties need eachother to compute a specific value.

```typescript
export class CompanyDetailComponent{
  // input state subjects
  private currentSiteId$$ = new ReplaySubject<string>(1);
  private sites$$ = new ReplaySubject<any[]>(1);

  // input stream setters
  @Input() set currentSiteId(v: string){
    if(v){ // we don't care about null values in this case
      this.currentSiteId$$.next(v);
    }
  };
  @Input() set sites(v: any[]){
    if(v){ // we don't care about null values in this case
      this.sites$$.next(v);
    }
  };
}
```

*Note: We use the `$$` suffix so we can see the observable is a `Subject`.*

We have now created observables from these `@Input()` properties. I have created the library [ngx-reactivetoolkit](https://www.npmjs.com/package/ngx-reactivetoolkit) to make this even easier. We can see an example on how this cleans up the code right below:

```typescript
export class CompanyDetailComponent{
  @Input() currentSiteId: string
  @Input()  sites: any[]
  @Changes('currentSiteId') currentSiteId$;
  @Changes('sites') sites$;
}
```

For this article we will continue with the native approach but be sure to check it out if you find the time. The toolkit might also gave some other good stuff that could help you.

Moving on... Remember that we mentioned that everything could be a stream? This also means our template events can be linked to streams.

```html
<button [disabled]="previousDisabled$|async" (click)="previousClicked()">
  Previous site
</button>
<button [disabled]="nextDisabled$|async" (click)="nextClicked()">
  Next site
</button>
```

```typescript
  // this will be used to communicate with the siteChanged @Output()
  private nav$$ = new Subject<number>();

  previousClicked(): void {
    this.nav$$.next(-1); // decrement
  }
  
  nextClicked(): void {
    this.nav$$.next(+1); // increment
  }
```

The next thing that we want to do is determine the presentation streams and the Output streams. For the presentation streams we can simply have a look at the template:

```html
<button [disabled]="previousDisabled$|async" (click)="previousClicked()">
  Previous site
</button>
<button [disabled]="nextDisabled$|async" (click)="nextClicked()">
  Next site
</button>
{{ currentSiteNumber$|async }}/ {{totalSites$|async}}
<ng-container *ngIf="currentSite$|async as currentSite">
  ...
</ng-container>

```

When looking at this template we can easily determine the presentation streams. These are all the streams that use an `async` pipe.

- `previousDisabled$`
- `nextDisabled$`
- `currentSiteNumber$`
- `totalSites$`
- `currentSite$`

The only `@Output()` stream for this template is for the `siteChanged` `@Output()`. Did you know an `EventEmitter` is an observable behind the scenes? Well, it is and we can replace the `EventEmitter` with any kind of observable.

This means that this code:

```typescript
@Output() siteChanged = new EventEmitter();
```

can be refactored to:

```typescript
@Output() siteChanged = this.siteChanged$;
```

Again, for this example the solution might be personal preference but when the flows become more complex we believe this approach can really shine! Think about a search component that looks like this...

```typescript
@Output() search$ = this.searchControl.valueChanges$.pipe(
    debounceTime(100),
    distinctUntilChanged()
)
```

This can become very powerful very fast!

The next thing that we want to do is calculate intermediate private streams. If you don't know what we mean with that, please check out the article on the [SIP principle](https://blog.strongbrew.io/the-sip-principle/). We use these intermediate streams to calculate the presentation streams, which are the streams that we will use in our template, and in the `@Output()` streams

The `currentIndex$` stream and the `indexWithSites$` stream are streams that are being used to calculate the presentationstreams. The `siteChanged$` stream will be used by the `siteChanged` `@Output()` property later on
We can see the calculation in this example:

```typescript
// intermediate streams
// the current index, calculated by the current site id and the sites
private currentIndex$ = combineLatest([this.currentSiteId$$, this.sites$$])
  .pipe(
    map(([currentSiteId, sites]) => 
      sites?.map(site => site?.id).indexOf(currentSiteId)    
    )
  );

// an array that always contains the currentIndex and all the sites
private indexWithSites$ = combineLatest([this.currentIndex$, this.sites$$]);

// every time the nav button is clicked, we need to calculate the id that
// needs to be emitted to the siteChanged @Output()
private siteChanged$ = this.nav$$.pipe(
  withLatestFrom(this.sites$$, this.currentIndex$),
  map(([navigationIndex, sites, currentIndex]) => 
    sites[currentIndex + navigationIndex]?.id
  )
)
```

Now we can start by implementing the  presentation streams and the `@Output()` stream.
In the code snippet below we can see the complete code: You can also check the [StackBlitz example](https://stackblitz.com/edit/reactive-dumb-components-reactive?file=src/app/components/company-detail/company-detail.component.ts)


```typescript

@Component({
  selector: 'app-company-detail',
  template: `
    <button [disabled]="previousDisabled$|async" (click)="previousClicked()">
      Previous site
    </button>
    <button [disabled]="nextDisabled$|async" (click)="nextClicked()">
      Next site
    </button>
    {{ currentSiteNumber$|async }}/ {{totalSites$|async}}
    <ng-container *ngIf="currentSite$|async as currentSite">
      <h2>{{currentSite.name}}</h2>
      <p>Address: {{currentSite.address}}</p>
    </ng-container>

  `,
  styleUrls: ['./company-detail.component.css']
})
export class CompanyDetailComponent{
  // local state subjects and input state subjects
  private nav$$ = new Subject<number>();
  private currentSiteId$$ = new ReplaySubject<string>(1);
  private sites$$ = new ReplaySubject<any[]>(1);

  // input stream setters
  @Input() set currentSiteId(v: string){
    if(v){
      this.currentSiteId$$.next(v);
    }
  };
  @Input() set sites(v: any[]){
    if(v){
      this.sites$$.next(v);
    }
  };

  // intermediate streams
  private currentIndex$ = combineLatest([this.currentSiteId$$, this.sites$$])
    .pipe(
      map(([currentSiteId, sites]) => 
        sites?.map(site => site?.id).indexOf(currentSiteId)    
      )
    );
  private indexWithSites$ = combineLatest([this.currentIndex$, this.sites$$]);

  private siteChanged$ = this.nav$$.pipe(
    withLatestFrom(this.sites$$, this.currentIndex$),
    map(([navigationIndex, sites, currentIndex]) => 
     sites[currentIndex + navigationIndex]?.id
    )
  )

  // output streams and presentational streams
  @Output() siteChanged = this.siteChanged$;
  previousDisabled$ = this.currentIndex$.pipe(
    map(currentIndex =>currentIndex === 0)
  )
  nextDisabled$ = this.indexWithSites$.pipe(
    map(([currentIndex, sites]) => currentIndex === sites?.length -1)
  )
  currentSite$ = this.indexWithSites$.pipe(
    map(([currentIndex, sites]) => sites[currentIndex])
  )
  totalSites$ = this.sites$$.pipe(
    map(sites => sites?.length)
  )
  currentSiteNumber$ = this.currentIndex$.pipe(
    map(v => v + 1)
  )

  previousClicked(): void {
    this.nav$$.next(-1);    
  }
  
  nextClicked(): void {
    this.nav$$.next(+1);    
  }
}

```

## Conclusion

We can make components completely reactive if we want to by:
- Binding other observables to `@Output()` properties
- Using setters to populate `@Input()` properties
- We can use `ngx-reactivetoolkit` to make the code cleaner

For simple component this can be seen as overkill, but it could definitely help when dumb components become more complex and rely of different streams of data that are happening asynchronously.
