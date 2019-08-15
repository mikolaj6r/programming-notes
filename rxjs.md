# RxJS

## Properly handling errors using `catchError`

So you've been using `catchError` just treating it like `.catch` in a promise-base API and it all seemed all good and sweet. But sometimes you encountered a bug where a stream would not be called again after an error. _But you caught the error with `catchError` and returned a new stream_, what could go wrong?🤔. Well just know that:
**`catchError` replaces whole stream, WHOLE STREAM**. Now lets see an example:

```typescript
source$.pipe(
  // switchMap can fail
  switchMap(something => from(resourceGetterFn(something))).pipe(
    // resolveResourceResponse can fail
    mergeMap(response => resolveResourceResponse(response))
  )
);
```

Now, what would happen when we did this:

```typescript
source$.pipe(
  // previous code with switchMap etc
  catchError(_ => {
    return of();
  })
);
```

So, error is propagated and is caught by `catchError`, that's all and good. But again **CATCH ERROR REPLACES WHOLE STREAM!**(and we are returning an empty Observable). That means, **after an error, that operator is just an empty Observable**.

### Solution

Solution would be... well reading the docs and such (and actually understanding what code you are writing). To solve this problem we just need to move `catchError` **inside switchMap**.

```typescript
source$.pipe(
  // switchMap can fail
  switchMap(something => from(resourceGetterFn(something))).pipe(
    mergeMap(response => resolveResourceResponse(response)),
    catchError(_ => {
      return of();
    })
  )
);
```

There, no magic, no weird copy-paste from stack. That's all.

_Reference: [this great article](https://medium.com/city-pantry/handling-errors-in-ngrx-effects-a95d918490d9)_

## Recipes

### Pooling

When you want to update stuff every X seconds

```js
const timer$ = timer(0, 5000);

timer$.pipe(
  exhaustMap(() => /* your http call */)
)
```

`exhaustMap` will not fire another request till previous request in-flight is not finished. If you want to drop that request and start another one you probably need `switchMap`.

### Drag and Drop

```js
const element = querySelector('element');

const mouseDown$ = fromEvent(element, 'mousedown');
const mouseUp$ = fromEvent(element, 'mouseup');
const mouseLeave$ = fromEvent(element, 'mouseleave');
const mouseMove$ = fromEvent(element, 'mousemove');

const stop$ = merge(mouseLeave$, mouseUp$);

const dragAndDrop$ = mouseDown$.pipe(
  exhaustMap(() => mouseMove$.pipe(takeUntil($stop)))
);
```

### Prevent double click

This is very useful :).
[A **very basic** implementation could be found here.](https://codesandbox.io/s/old-feather-rz6xm)

```js
const button = querySelector('element');
const buttonClick$ = fromEvent(button, 'click');

const preventDoubleSubmit$ = buttonClick$.pipe(
  exhaustMap(() => http.post(/* something */))
);
```

## Uncommon Types

### Notification

Aside from all the `Subject`-y related types there is also notification.

`Notification` **does not create an observer**. It wraps it annotating it with additional metadata.

Example:

```js
of(1)
  .pipe(mapTo(new Notification('E')))
  .subscribe(console.log);
/*
    Notification {kind: "E", value: undefined, error: undefined, hasValue: false, constructor: Object}
    kind: "E"
    value: undefined
    error: undefined
    hasValue: false
    <constructor>: "Notification"
*/
```

As you see it can swallow original values. Notification can be of a different type (type corresponds to observable life cycle).

## Uncommon Operators

### Materialize / dematerialize

So you know about `Notification` type. It's all good and great but you probably wonder how you could use it.

Lets say you have a source that can error out. Sure, that could happen but when that does happen, **error bubbles up and ignores every operator that is yet to come**. This might be a problem.

To prevent this you might want to turn the error into `Notification` using `materialize` operator. If a given source errors out you can check if `Notification` is of type error and act accordingly without skipping operators.

Example:

```js
// sample stream
interval(500)
  .pipe(
    mapTo('normal value'),
    // sometimes value, sometimes throw
    map(v => {
      if (randomInt() > 50) {
        throw new Error('boom!');
      } else return v;
    }),
    materialize(),
    // turns Observable<T> into Notification<Observable<T>>
    // so we can delay or use other operators.
    delay(500),
    // Notification of value (error message)
    map(n => (n.hasValue ? n : new Notification('N', n.error.message, null))),
    // back to normal
    dematerialize()
  )
  // now it never throw so in console we will have
  // `normal value` or `boom!` but all as... normal values (next() emission)
  // and delay() works as expected
  .subscribe(v => console.log(v));
```