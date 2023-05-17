# Angular Zoneless

Angular 16 introduces a new and better server side rendering with hydration support.

To render server-side without the hassle of using Zone.js, we need a custom Zone.js implementation
that works different from hooking into the whole environment (like setTimeout, async to generator, etc).

Why is Zone.js problematic?

 - It hooks into the whole environment, which means it hooks into setTimeout, setInterval, etc.
 - It hooks into async/await and converts it to generators, because Zone.js does not support native async/await.
 - It does not support async lifecycle hooks like ngOnInit, which means you can't use async/await in ngOnInit.
 - It makes everything slower and pollutes the stack trace with Zone.js internals.
 - It forces you do compile the whole application with all dependencies via Angulars compiler (to replace async with generators),
   making it unnecessary complex and hard to replace providers in the server side rendering process.

I prefer to have my Node.js instance clean and not polluted with Zone.js, so I can use async/await
without any problems. This package provides such a Zone.js implementation, making SSR and hydration possible without
Zone.js. It comes with some limitations though.

It hooks automatically into all lifecycle hooks and makes Zone.js aware of them.
This means, you can use `async ngOnInit` and the SSR as well as hydration on the client wait
correctly until your init procedure is done (like loading data from the server).

## How to set up

```typescript
import { withZoneLessModule } from 'angular-zoneless';
import { ApplicationConfig } from '@angular/core';
import { provideClientHydration } from "@angular/platform-browser";

export const appConfig: ApplicationConfig = {
    providers: [
        provideClientHydration(),
        withZoneLessModule(),
        
        // ...
    ]
};
```

Or use `ZoneLessModule.forRoot()` for non-standalone applications.

## How to use

Then make sure to use `async` in your lifecycle hooks:

```typescript
@Component({
    selector: 'app-root',
    template: `
        <h1>{{ title }}</h1>
    `
})
export class AppComponent implements OnInit {
    title: string;
    
    async ngOnInit() {
        this.title = await this.getTitle();
    }
    
    async getTitle() {
        return 'Hello World';
    }
}
```

Once the Promise of `ngOnInit` is resolved in all components, the ZoneJS implementation calls onStable,
which allows hydration and SSR to finish.

This works since this module hooks into all async methods of your components. By doing that it knows
about all the called initial async methods and can wait for them to finish. Once finished, it calls
onStable and allows hydration and SSR to finish.

This works with activatedRoute as well, but make sure to use it in your ngOnInit method, not in the constructor:

```typescript

@Component({
    selector: 'app-root',
    template: `
        <h1>{{ title }}</h1>
    `
})
export class AppComponent implements OnInit {
    title: string;
    
    constructor(private activatedRoute: ActivatedRoute) {
    }
    
    ngOnInit() {
        this.activatedRoute.params.subscribe(params => {
            this.load(params.id);
        });
    }

    async load(id: string) {
        this.title = await this.getTitle(id);
    }
}
```

If you render anything dynamic like (click)="load()" and `load` is async, you need either use RxJS with async pipe to render dynamically,
or make sure `load` calls `ChangeDetectorRef.detectChanges()` once the data is loaded.

Soon, you also will be able to use signals to make this easier.

```typescript
@Component({
    selector: 'app-root',
    template: `
        <h1>{{ title }}</h1>
        <button (click)="load()">Load</button>
    `
})
export class AppComponent implements OnInit {
    title: string;

    constructor(private cd: ChangeDetectorRef) {
    }

    async ngOnInit() {
        this.title = await this.getTitle();
    }

    async getTitle() {
        return 'Hello World';
    }

    async load() {
        this.title = await this.getTitle();
        this.cd.detectChanges();
    }
}
```
