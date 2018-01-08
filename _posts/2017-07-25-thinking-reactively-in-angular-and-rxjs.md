---
layout: post
title: Thinking reactively in Angular and RXJS
published: true
author: brechtbilliet
description: RxJS is an awesome library that can help us with creating reactive web applications. Reactive web applications can be overwhelming in the beginning, but eventually, they can be really rewarding.This article is all about making the paradigm switch from thinking imperatively towards thinking reactively.
comments: true
date: 2017-07-25
subclass: 'post'
categories: 'casper'
disqus: true
tags: Angular, RxJS
---

## Foreword

RxJS is an awesome library that can help us with creating **reactive web applications**. Reactive web applications can be overwhelming in the beginning, but eventually, they can be really rewarding.

This article is all about making the paradigm switch from thinking imperatively towards **thinking reactively**.
In this article, we will explain how to write a reactive calendar application in only a few lines of code (**spoiler: It's gonna be real time too**).

We will use Angular, Angular Material, TypeScript, RxJS, Firebase, and AngularFire as our main technology stack. Keep in mind that this article really focusses on reactive programming. Don't expect a deep dive into all RxJS operators, but rather expect an explanation of how to draw, think, and reason about reactive web applications. We will learn **how to think in streams**. If you haven't heard of streams yet, please read [this awesome article](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) first.

**Note:** This article contains personal terminology.

## The Reactive Calendar
This is the application we are going to write. It's a small but complete calendar application that allows us to:

- Switch between different view modes: day, week, month.
- Navigate to previous and next days, weeks, and months.
- Add, update, and remove appointments.
- Search for specific appointments.

![Reactive calendar](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar1.png)


The user can interact with the following UI elements:

- **Next button:** Allows the user to go to the next day in day mode, week in week mode, etc.
- **Previous button:** Allows the user to go to the previous day in day mode, week in week mode, etc.
- **Day, week, month buttons:** Allows the user to switch between the different view modes
- **Search term input:** Allows the user to filter the appointments on the fly
- **Plus-buttons in the grid:** Allows the user to create new appointments
- **Trashcan buttons in the grid:** Allows the user to remove appointments
- **Description inputs:** Allows the user to update the description of an appointment

I decided to use Firebase as a backend and because of that, our application will be realtime and offline first by default!

**Note:** One small issue, I've been a bit lazy so we can only create lunch appointments. =) But hey! Consider it some homework.


## Setting Up the Project

I've created the git branch **initial** to get us started. It contains the default logic/components, setup, and styles. There is no reactive code written yet, just plain Angular code. The goal is to write the reactive part ourselves.


### The Component Tree

![The component tree](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar3.png)
The dumb components (blue) are already implemented. The ```app-root``` (orange) is the one and only smart component in the application and the only place where we will write code.

If you don't know the difference between smart and dumb components, [read this first](http://blog.brecht.io/components-demystified/#smart-vs-dumb-components).


### Installing the Project Locally
First of all, we have to clone the project locally and check out the **initial** branch. This branch already contains all the uninteresting parts that don't have anything to do with this article. 

In the terminal, we have to go to the folder where we want to install the project and run the following commands:

```
$ git clone git@github.com:brechtbilliet/reactive-calendar.git
$ cd reactive-calendar/reactive-calendar
$ git checkout initial
$ npm install
```

### Setting Up Firebase

We are using [Firebase](https://firebase.google.com/) as our backend because it requires minimal setup, it's realtime by default, and [AngularFire](https://github.com/angular/angularfire2) gives us streams for free. We can complete the Firebase configuration in a few steps:

- Go to [https://firebase.google.com](https://firebase.google.com/), click on the "GO TO CONSOLE" button, and choose your Google account.
- Click on the "Add project" button and choose a name for your project. Let's take **"reactive-calendar"** to keep it simple. 
- Click on the "CREATE PROJECT" button. Now we should be redirected to [something like this](https://console.firebase.google.com/project/reactive-calendar/overview).
- In the Authentication tab, go to "SIGN-IN METHOD" and enable the "Anonymous" setting.
- Click on database and navigate to the rules tab. Set the read and write property to "true" and click "publish": 
```json
{
    "rules": {
        ".read": "true",
        ".write": "true"
    }
}
```
- Go back to the overview by clicking on the home icon, and then select "Add Firebase to your web app".
- Copy the config with the correct properties and replace the firebaseConfig object in src/app/app.module.ts with these properties.
It might look something like this:

```typescript
const firebaseConfig = {
    apiKey: "AIzaSyBuqjTJd5v6xTf8D2EZmvFUl8lseH8lVuHU",
    authDomain: "reactive-calendar.firebaseapp.com",
    databaseURL: "https://reactive-calendar.firebaseio.com",
    projectId: "reactive-calendar",
    storageBucket: "reactive-calendar.appspot.com",
    messagingSenderId: "3978123451455750"
};
```

Let's continue. Start the project by running the following command and open your browser on [http://localhost:4200](http://localhost:4200). 


```
npm start
```

As you can see, this just handles static data, the buttons/inputs won't work, and the appointments are not loaded yet.
This is where we start from.

## Thinking Reactively

Now comes the tricky part. We are trying to forget imperative programming for now, and we are trying to evolve into a reactive mindset.

### Marble Diagrams

To be able to think reactively, we need some kind of graphic model so we can picture streams in our head. Marble diagrams are a great way to do that.
As you can see in the image below, a marble represents a value over time.

![Marble diagrams](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar12.png)

The website [rxmarbles.com](http://rxmarbles.com/) has a great playground for learning how to use and draw marble diagrams.

#### ASCII Documentation

One could argue that code should not be documented and be self-explanatory. I don't believe that to be the case when writing complex streams. When we document complex streams, we can see what's going on inside the stream, which makes it easier for our colleagues.
Streams can be documented by ASCII documentation. Since that is not really part of this article, I'm only going to show a small example below.

```typescript
// a$ gets three values over time and then stops
// a$: -------a-----b-----c|

// b$ has an initial value (a), has three values in total
// and will keep on living
// b$: a------b-----c------...
```

### Imperative Programming: What Does the App Have to Do?

When we think about the functionality of our application, we quickly notice that there are quite a few corner cases and special scenarios. For every interaction the user makes in the UI, the app needs to handle that specific interaction accordingly. Sometimes it has to combine these interactions and handle that specific combination as well. Take this crazy (but simple) example, for instance.

<blockquote>
When the view mode is changed to week mode, and the previous view mode was month mode, and the month was June, and the year was 2017, and an appointment was added, while the search term was set to "Brecht", then we would have to update...
</blockquote>

Yes, we would have to update a bunch of stuff. This is imperative thinking, and it can become exhausting. There is a big chance that we forget certain corner cases. Let's not even imagine that we have to combine that with asynchronous actions as well.

In the image below, we see all the different interactions the user has in the calendar application.
![Application events](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar2.png)

As we can see, for every specific interaction, the UI will have to update specific things.

### Reactive programming: What Data Will Change, and What Data Do the Components Need?

#### Source Streams

Now, let's completely stop with what we are thinking. Let's free our minds and stop thinking about corner cases and special scenarios. We have to learn to think in streams. A stream is a collection of events that will change over time. Think about what can change in your application and call these streams of data. Let's call them **source streams**. 

**Note:** For readability purposes, we will suffix all the streams with a ```$``` symbol.

We can come up with 4 source streams:

- **navigation$:** Can contain the values: -1, 0 or 1
- **viewMode$:** Can contain the values: DAY, WEEK, or MONTH
- **searchTerm$:** The value of the search field
- **appointments$:** This is an array of appointments that comes from Firebase

![data streams](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar4.png)

That was pretty easy. We just had to think about the events that can occur in our application. A user can navigate, change view mode, search for appointments, and the appointments in Firebase can change. This is the beginning of thinking reactively. Don't think about who triggers what. Think about the changes as streams.


It's always a good idea to draw marble diagrams to make it easier to reason.

![data stream diagram](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar5.png)

```The appointments$``` is a stream that will be provided to us by AngularFire, but the ```viewMode$```, ```searchTerm$```, and ```navigation$``` are simple behavior subjects. We use subjects because we need to control the values of the streams ourselves, and we use the ```BehaviorSubject``` in particular because all our source streams need an initial value.

```typescript
export class AppComponent {
    ...
    // this is how we can retrieve the list of appointments from angularfire
    appointments$ = this.db.list('/appointments');
     // 0--------(+1)----(+1)----(-1)-------------...
    viewMode$ = new BehaviorSubject(VIEW_MODE.MONTH);
    navigation$ = new BehaviorSubject(0);
    searchTerm$ = new BehaviorSubject('');

	// because we set up the angularfire configuration correctly, we can just
	// inject the angularfiredatabase right here and use it
    constructor(private db: AngularFireDatabase) {
    }
    ...
}

```

These subjects get values from the simple interactions from the template.

```typescript
@Component({
    ...
    template: `
        <topbar
                (next)="onNext()"
                (previous)="onPrevious()"
                (setViewMode)="onSetViewMode($event)"
                (searchChanged)="onSearchChanged($event)">
        </topbar>
        ...
    `
})
export class AppComponent {
    ...
    
    onSetViewMode(viewMode: string): void {
        // when the viewmode changes, update its subject
        this.viewMode$.next(viewMode);
    }

    onPrevious(): void {
        // when the user clicks the previous button
        // update the navigation subject
        this.navigation$.next(-1);
    }

    onNext(): void {
        // when the user clicks the next button
        // update the navigation subject
        this.navigation$.next(1);
    }

    onSearchChanged(e: string): void {
        // when the user searches
        // update the searchterm subject
        this.searchTerm$.next(e);
    }
}

```

#### Presentational Streams

Now we have to think about the data that our components need, because those components will need to be updated based on those source streams.
Let's take this code sample, for instance: 

```html
<div [ngSwitch]="XX" class="main">
    <day-view
            *ngSwitchCase="'DAY'"
            [appointments]="XX"
            [date]="XX"
            ...>
    </day-view>
    <week-view
            *ngSwitchCase="'WEEK'"
            [appointments]="XX"
            [year]="XX"
            [week]="XX"
            ...>
    </week-view>
    <month-view
            *ngSwitchCase="'MONTH'"
            [month]="XX"
            [year]="XX"
            [appointments]="xxx"
            ...>
    </month-view>
</div>
```

I marked the input properties with XX to show what our components need in terms of data. These places will need streams as well. Let's call them **presentational streams**.

Let's try to fill in these gaps, shall we?

**Note:** We use the [async pipe](https://angular.io/api/common/AsyncPipe) from Angular to subscribe/unsubscribe the streams automatically.

```html
<div [ngSwitch]="viewMode$|async" class="main">
    <day-view
            *ngSwitchCase="'DAY'"
            [appointments]="filteredAppointments$|async"
            [date]="currentDate$|async"
            ...>
    </day-view>
    <week-view
            *ngSwitchCase="'WEEK'"
            [appointments]="filteredAppointments$|async"
            [year]="currentYear$|async"
            [week]="currentWeek$|async"
            ...>
    </week-view>
    <month-view
            *ngSwitchCase="'MONTH'"
            [month]="currentMonth$|async"
            [year]="currentYear$|async"
            [appointments]="filteredAppointments$|async"
            ...>
    </month-view>
</div>
```

We have gathered the 6 following presentational streams:

- **viewMode$ (string):** needed to determine which view has to be shown
- **filteredAppointments$ (Array < Appointment >):** needed by day view, week view, and month view to render the correct appointments
- **currentDate$ (date):** the current date for the day view
- **currentWeek$ (number):** the current week for the week view
- **currentYear$ (number):** needed by week view and month view
- **currentMonth$ (number):** needed by the month view

Okay, great, we know the source streams, which are the sources of change in our application.
We know the presentational streams, which are simply the streams that our components need. Now it's time for the cool part: **We need to create those presentational streams based on the source streams**.

![sources to presentational streams](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar6.png)

The first presentational stream we need is ```viewMode$```. This is already an easy one, since ```viewMode$``` is also a source stream.

#### currentDate$
![currentDate$](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar7.png)

**Note:** We use moment.js for date calculation. The suffix M after the currentDate property shows that the type is ```Moment```. So in short, it's not just a date, but a moment wrapper.

```typescript
// we will need this stream a few times, so let's extract the stream 
// in a currentDateM first

// viewMode$:     M------------------W---------------D--------...
// navigation$:   0---(+1)-(-1)----------(+1)-(-1)------------...
// currentDateM$: d---d----d---------d---d----d------d--------...
private currentDateM = this.viewMode$.flatMap((viewMode: string) => {
    // every time the viewMode changes, the navigation should be reset as well
    // the dateM variable will contain the navigation and because of the 
    // flatMap it will reset every time the view mode changes
    // if the navigation$ changes afterwards it will manipulate the dateM object
    // by adding months, weeks, or days depending on the viewMode
    const dateM = moment();
    return this.navigation$
        .map((action: number) => {
            switch (viewMode) {
                case VIEW_MODE.MONTH:
                    return dateM.startOf('month').add(action, "months");
                case VIEW_MODE.WEEK:
                    return dateM.startOf('week').add(action, "weeks");
                case VIEW_MODE.DAY:
                    return dateM.startOf('day').add(action, "days");
            }
            return dateM;
        })
})
currentDate$ = this.currentDateM$.map(dateM => dateM.toDate());

```

#### currentWeek$

Based on the ```currentDateM$``` we can calculate the current week. The ```currentDateM$``` is just a moment object of the current date based on the navigation and viewMode.

![currentWeek$](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar8.png)

```typescript
currentWeek$ = this.currentDateM$.map(dateM => dateM.week());
```

#### currentMonth$

Just like we calculated the ```currentWeek$``` based on the ```currentDateM$```, we can do the same thing here.

![currentMonth$](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar9.png)

```typescript
currentMonth$ = this.currentDateM$.map(dateM => dateM.month());
```
#### currentYear$

Just like we calculated the ```currentWeek$``` and the ```currentMonth$``` based on the ```currentDateM$```, we can do the same thing here.

![currentYear$](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar10.png)

```typescript
currentYear$ = this.currentDateM$.map(dateM => dateM.year());
```

#### filteredAppointments$

This is the most important stream. It is used to show the appointments in all the different views, and it is dependent on a bunch of streams:

- viewMode$
- currentDateM$
- appointments$
- searchTerm$

This looks a bit more complex, but let's give it a go. 

**Note:** the ```[]``` in the image below stands for an empty array, the ```[.]``` for an array with one value, and so on.

![filteredAppointment$](https://raw.githubusercontent.com/brechtbilliet/brechtbilliet.github.io/master/_posts/reactivecalendar/reactivecalendar11.png)

Let's take the time to process this image. The operator we will use to combine all these streams is called **combineLatest**. It will create a stream that will wait until all streams have a value and will start emitting values for every change of every stream. 

So basically, it gives us a function where we have all the information we need. The appointments in Firebase, the view mode, the search term, and the current date. Based on those values, we can calculate the appointments for every view:

```typescript 
filteredAppointments$ = Observable.combineLatest(
    [this.viewMode$, this.currentDateM$, 
    this.appointments$, this.searchTerm$],
    (viewMode: string, currentDateM: Moment, 
        appointments: Array<Appointment>, searchTerm: string) => {
        switch (viewMode) {
            // calculate the appointments for the month-view based on
            // the current date, the appointments in firebase 
            // and the searchterm
            case VIEW_MODE.MONTH:
                return appointments
                    .filter(item => moment(item.date).format('MM/YYYY') === currentDateM.format('MM/YYYY'))
                    .filter(item => this.filterByTerm(item, searchTerm));
             // calculate the appointments for the week-view based on
             // the current date, the appointments in firebase
             // and the searchterm
            case VIEW_MODE.WEEK:
                return appointments
                    .filter(item => moment(item.date).format('ww/YYYY') === currentDateM.format('ww/YYYY'))
                    .filter(item => this.filterByTerm(item, searchTerm));
            // calculate the appointments for the day-view based on
            // the current date, the appointments in firebase
            // and the searchterm
            case VIEW_MODE.DAY:
                return appointments
                    .filter(item => moment(item.date).format('DD/MM/YYYY') === currentDateM.format('DD/MM/YYYY'))
                    .filter(item => this.filterByTerm(item, searchTerm));

        }
    });

private filterByTerm(appointment: Appointment, term: string): boolean {
    return appointment.description.toLowerCase().indexOf(term.toLowerCase()) > -1;
}
```

This is all we have to do in order to create a kick-ass realtime reactive calendar application. We have created it in no time and with only a few lines of code. If we think about it, we will soon realize that all corner cases have been covered.

## Performance Improvements

The complete component looks like the code snippet below now. The calendar should be completely functional in your browser.

```typescript
import { Component } from '@angular/core';
import { VIEW_MODE } from '../../constants';
import * as moment from 'moment';
import { Appointment } from '../../types/appointment.type';
import { AngularFireDatabase } from 'angularfire2/database';
import { Observable } from 'rxjs/Observable';
import Moment = moment.Moment;
import { BehaviorSubject } from 'rxjs/BehaviorSubject';


@Component({
    selector: 'app-root',
    template: `
        <topbar
                (next)="onNext()"
                (previous)="onPrevious()"
                (setViewMode)="onSetViewMode($event)"
                (searchChanged)="onSearchChanged($event)">
        </topbar>
        <div [ngSwitch]="viewMode$|async">
            <day-view
                    *ngSwitchCase="VIEW_MODE.DAY"
                    [appointments]="filteredAppointments$|async"
                    [date]="currentDate$|async"
                    (removeAppointment)="onRemoveAppointment($event)"
                    (addAppointment)="onAddAppointment($event)"
                    (updateAppointment)="onUpdateAppointment($event)"
            >
            </day-view>
            <week-view
                    *ngSwitchCase="VIEW_MODE.WEEK"
                    [appointments]="filteredAppointments$|async"
                    [year]="currentYear$|async"
                    [week]="currentWeek$|async"
                    (removeAppointment)="onRemoveAppointment($event)"
                    (addAppointment)="onAddAppointment($event)"
                    (updateAppointment)="onUpdateAppointment($event)"
            >
            </week-view>
            <month-view
                    *ngSwitchCase="VIEW_MODE.MONTH"
                    [month]="currentMonth$|async"
                    [year]="currentYear$|async"
                    [appointments]="filteredAppointments$|async"
                    (removeAppointment)="onRemoveAppointment($event)"
                    (addAppointment)="onAddAppointment($event)"
                    (updateAppointment)="onUpdateAppointment($event)"
            >
            </month-view>
        </div>
    `,
    styleUrls: ['./app.component.less']
})
export class AppComponent {
    VIEW_MODE = VIEW_MODE;
    viewMode$ = new BehaviorSubject(VIEW_MODE.MONTH);
    // 0--------(+1)----(+1)----(-1)-------------...
    navigation$ = new BehaviorSubject<number>(0);
    searchTerm$ = new BehaviorSubject('');

    // -----MONTH---------------------YEAR------...
    // -----MONTH-------------------------------...
    // -----(d)---------------------------------...
    // --------(+1)----(+1)----(-1)-------------...
    // -----d---d-------d-------d-----d----------...

    private currentDateM$ = this.viewMode$.flatMap((viewMode: string) => {
        let dateM = moment();
        return this.navigation$
            .map((action: number) => {
                switch (viewMode) {
                    case VIEW_MODE.MONTH:
                        return dateM.startOf('month').add(action, 'months');
                    case VIEW_MODE.WEEK:
                        return dateM.startOf('week').add(action, 'weeks');
                    case VIEW_MODE.DAY:
                        return dateM.startOf('day').add(action, 'days');
                }
                return dateM;
            })
    });

    currentDate$ = this.currentDateM$.map(dateM => dateM.toDate());
    currentYear$ = this.currentDateM$.map(dateM => dateM.year());
    currentMonth$ = this.currentDateM$.map(dateM => dateM.month());
    currentWeek$ = this.currentDateM$.map(dateM => dateM.week());
    appointments$ = this.db.list('/appointments');
    filteredAppointments$ = Observable.combineLatest([this.viewMode$, this.currentDateM$, this.appointments$, this.searchTerm$],
        (viewMode: string, currentDateM: Moment, appointments: Array<Appointment>, searchTerm: string) => {
            switch (viewMode) {
                case VIEW_MODE.MONTH:
                    return appointments
                        .filter(item => moment(item.date).format('MM/YYYY') === currentDateM.format('MM/YYYY'))
                        .filter(item => this.filterByTerm(item, searchTerm));
                case VIEW_MODE.WEEK:
                    return appointments
                        .filter(item => moment(item.date).format('ww/YYYY') === currentDateM.format('ww/YYYY'))
                        .filter(item => this.filterByTerm(item, searchTerm));
                case VIEW_MODE.DAY:
                    return appointments
                        .filter(item => moment(item.date).format('DD/MM/YYYY') === currentDateM.format('DD/MM/YYYY'))
                        .filter(item => this.filterByTerm(item, searchTerm));

            }
        });

    constructor(private db: AngularFireDatabase) {
    }

    private filterByTerm(appointment: Appointment, term: string): boolean {
        return appointment.description.toLowerCase().indexOf(term.toLowerCase()) > -1;
    }

    onSetViewMode(viewMode: string): void {
        this.viewMode$.next(viewMode);
    }

    onPrevious(): void {
        this.navigation$.next(-1);
    }

    onNext(): void {
        this.navigation$.next(1);
    }

    onSearchChanged(e: string): void {
        this.searchTerm$.next(e);
    }

    onRemoveAppointment(id: string): void {
        this.appointments$.remove(id);
    }

    onAddAppointment(date: Date): void {
        this.appointments$.push(new Appointment(date.toDateString(), ''));
    }

    onUpdateAppointment(appointment: Appointment): void {
        this.db.object('appointments/' + appointment.$key).set({
            description: appointment.description,
            date: appointment.date
        });
    }
}

```

There is only one problem. We use the same observables multiple times in our template. Since observables are cold by default, they will get executed every time there is a subscription. In Angular, this means a subscription for every async pipe. For performance reasons, we only want to recalculate these streams when something actually changes. For that purpose, we can try to use the ```share()``` operator from RxJS. The ```share()``` operator is an alias for ```publish().refCount()``` and will share the subscription.

However, that creates some problems with Angular and its async pipe.
The situation of the problem goes like this:

- Since we are using BehaviorSubjects, the streams will get an initial value (which is what we want, of course).
- The share() operator will emit that value on the first subscription
- When the app is initialized, the async pipes will start subscribing to the stream.
- Because the first async pipe triggered the first emit the rest of the async pipes will miss that value.

**Solution: shareReplay() will emit those values but keep track of them. That way, the async pipes will never miss a value.**
 

## Conclusion

We have created a completely reactive calendar that is performant and fixes a bunch of corner cases in only a few lines of code. Just by thinking about source streams and presentational streams, it wasn't even that hard. I hope that I can encourage more people to take on this reactive approach and start writing kick-ass applications.

## Special Thanks

I would like to give special thanks to the awesome people that reviewed this post and gave me pointers:

- Dominic Elm ([@elmd_](https://twitter.com/elmd_))
- Manfred Steyer ([@manfredsteyer](https://twitter.com/manfredsteyer))
- David Müllerchen ([@webdave_de](https://twitter.com/webdave_de))
- Maxim Robert ([@sizerOne](https://twitter.com/sizerone))

Thanks, guys! It means a lot!
