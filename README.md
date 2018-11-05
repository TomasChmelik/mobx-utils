# MobX-utils

_Utility functions and common patterns for MobX_

[![Build Status](https://travis-ci.org/mobxjs/mobx-utils.svg?branch=master)](https://travis-ci.org/mobxjs/mobx-utils)
[![Coverage Status](https://coveralls.io/repos/github/mobxjs/mobx-utils/badge.svg?branch=master)](https://coveralls.io/github/mobxjs/mobx-utils?branch=master)
[![Join the chat at https://gitter.im/mobxjs/mobx](https://badges.gitter.im/mobxjs/mobx.svg)](https://gitter.im/mobxjs/mobx?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

This package provides utility functions and common MobX patterns build on top of MobX.
It is encouraged to take a peek under the hood and read the sources of these utilities.
Feel free to open a PR with your own utilities. For large new features, please open an issue first.

# Installation & Usage

NPM: `npm install mobx-utils --save`

CDN: <https://unpkg.com/mobx-utils/mobx-utils.umd.js>

`import {function_name} from 'mobx-utils'`

# API

<!-- Generated by documentation.js. Update this documentation by updating the source code. -->

## fromPromise

`fromPromise` takes a Promise and returns a new Promise wrapping the original one. The returned Promise is also extended with 2 observable properties that track
the status of the promise. The returned object has the following observable properties:

-   `value`: either the initial value, the value the Promise resolved to, or the value the Promise was rejected with. use `.state` if you need to be able to tell the difference.
-   `state`: one of `"pending"`, `"fulfilled"` or `"rejected"`

And the following methods:

-   `case({fulfilled, rejected, pending})`: maps over the result using the provided handlers, or returns `undefined` if a handler isn't available for the current promise state.
-   `then((value: TValue) => TResult1 | PromiseLike<TResult1>, [(rejectReason: any) => any])`: chains additional handlers to the provided promise.

The returned object implements `PromiseLike<TValue>`, so you can chain additional `Promise` handlers using `then`. You may also use it with `await` in `async` functions.

Note that the status strings are available as constants:
`mobxUtils.PENDING`, `mobxUtils.REJECTED`, `mobxUtil.FULFILLED`

Observable promises can be created immediately in a certain state using
`fromPromise.reject(reason)` or `fromPromise.resolve(value?)`.
The main advantage of `fromPromise.resolve(value)` over `fromPromise(Promise.resolve(value))` is that the first _synchronously_ starts in the desired state.

It is possible to directly create a promise using a resolve, reject function:
`fromPromise((resolve, reject) => setTimeout(() => resolve(true), 1000))`

**Parameters**

-   `promise` **IThenable&lt;T>** The promise which will be observed

**Examples**

```javascript
const fetchResult = fromPromise(fetch("http://someurl"))

// combine with when..
when(
  () => fetchResult.state !== "pending",
  () => {
    console.log("Got ", fetchResult.value)
  }
)

// or a mobx-react component..
const myComponent = observer(({ fetchResult }) => {
  switch(fetchResult.state) {
     case "pending": return <div>Loading...</div>
     case "rejected": return <div>Ooops... {fetchResult.value}</div>
     case "fulfilled": return <div>Gotcha: {fetchResult.value}</div>
  }
})

// or using the case method instead of switch:

const myComponent = observer(({ fetchResult }) =>
  fetchResult.case({
    pending:   () => <div>Loading...</div>,
    rejected:  error => <div>Ooops.. {error}</div>,
    fulfilled: value => <div>Gotcha: {value}</div>,
  }))

// chain additional handler(s) to the resolve/reject:

fetchResult.then(
  (result) =>  doSomeTransformation(result),
  (rejectReason) => console.error('fetchResult was rejected, reason: ' + rejectReason)
).then(
  (transformedResult) => console.log('transformed fetchResult: ' + transformedResult)
)
```

Returns **IPromiseBasedObservable&lt;T>** 

## isPromiseBasedObservable

Returns true if the provided value is a promise-based observable.

**Parameters**

-   `value`  any

Returns **[boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean)** 

## moveItem

Moves an item from one position to another, checking that the indexes given are within bounds.

**Parameters**

-   `target` **ObservableArray&lt;T>** 
-   `fromIndex` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** 
-   `toIndex` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** 

**Examples**

```javascript
const source = observable([1, 2, 3])
moveItem(source, 0, 1)
console.log(source.map(x => x)) // [2, 1, 3]
```

Returns **ObservableArray&lt;T>** 

## lazyObservable

`lazyObservable` creates an observable around a `fetch` method that will not be invoked
until the observable is needed the first time.
The fetch method receives a `sink` callback which can be used to replace the
current value of the lazyObservable. It is allowed to call `sink` multiple times
to keep the lazyObservable up to date with some external resource.

Note that it is the `current()` call itself which is being tracked by MobX,
so make sure that you don't dereference too early.

**Parameters**

-   `fetch`  
-   `initialValue` **T** optional initialValue that will be returned from `current` as long as the `sink` has not been called at least once (optional, default `undefined`)

**Examples**

```javascript
const userProfile = lazyObservable(
  sink => fetch("/myprofile").then(profile => sink(profile))
)

// use the userProfile in a React component:
const Profile = observer(({ userProfile }) =>
  userProfile.current() === undefined
  ? <div>Loading user profile...</div>
  : <div>{userProfile.current().displayName}</div>
)

// triggers refresh the userProfile
userProfile.refresh()
```

## fromResource

`fromResource` creates an observable whose current state can be inspected using `.current()`,
and which can be kept in sync with some external datasource that can be subscribed to.

The created observable will only subscribe to the datasource if it is in use somewhere,
(un)subscribing when needed. To enable `fromResource` to do that two callbacks need to be provided,
one to subscribe, and one to unsubscribe. The subscribe callback itself will receive a `sink` callback, which can be used
to update the current state of the observable, allowing observes to react.

Whatever is passed to `sink` will be returned by `current()`. The values passed to the sink will not be converted to
observables automatically, but feel free to do so.
It is the `current()` call itself which is being tracked,
so make sure that you don't dereference to early.

For inspiration, an example integration with the apollo-client on [github](https://github.com/apollostack/apollo-client/issues/503#issuecomment-241101379),
or the [implementation](https://github.com/mobxjs/mobx-utils/blob/1d17cf7f7f5200937f68cc0b5e7ec7f3f71dccba/src/now.ts#L43-L57) of `mobxUtils.now`

The following example code creates an observable that connects to a `dbUserRecord`,
which comes from an imaginary database and notifies when it has changed.

**Parameters**

-   `subscriber`  
-   `unsubscriber` **IDisposer**  (optional, default `NOOP`)
-   `initialValue` **T** the data that will be returned by `get()` until the `sink` has emitted its first data (optional, default `undefined`)

**Examples**

```javascript
function createObservableUser(dbUserRecord) {
  let currentSubscription;
  return fromResource(
    (sink) => {
      // sink the current state
      sink(dbUserRecord.fields)
      // subscribe to the record, invoke the sink callback whenever new data arrives
      currentSubscription = dbUserRecord.onUpdated(() => {
        sink(dbUserRecord.fields)
      })
    },
    () => {
      // the user observable is not in use at the moment, unsubscribe (for now)
      dbUserRecord.unsubscribe(currentSubscription)
    }
  )
}

// usage:
const myUserObservable = createObservableUser(myDatabaseConnector.query("name = 'Michel'"))

// use the observable in autorun
autorun(() => {
  // printed everytime the database updates its records
  console.log(myUserObservable.current().displayName)
})

// ... or a component
const userComponent = observer(({ user }) =>
  <div>{user.current().displayName}</div>
)
```

## toStream

Converts an expression to an observable stream (a.k.a. TC 39 Observable / RxJS observable).
The provided expression is tracked by mobx as long as there are subscribers, automatically
emitting when new values become available. The expressions respect (trans)actions.

**Parameters**

-   `expression`  
-   `fireImmediately` **[boolean](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean)** (by default false)

**Examples**

```javascript
const user = observable({
  firstName: "C.S",
  lastName: "Lewis"
})

Rx.Observable
  .from(mobxUtils.toStream(() => user.firstname + user.lastName))
  .scan(nameChanges => nameChanges + 1, 0)
  .subscribe(nameChanges => console.log("Changed name ", nameChanges, "times"))
```

Returns **IObservableStream&lt;T>** 

## StreamListener

## fromStream

Converts a subscribable, observable stream (TC 39 observable / RxJS stream)
into an object which stores the current value (as `current`). The subscription can be cancelled through the `dispose` method.
Takes an initial value as second optional argument

**Parameters**

-   `observable` **IObservableStream&lt;T>** 
-   `initialValue`  

**Examples**

```javascript
const debouncedClickDelta = MobxUtils.fromStream(Rx.Observable.fromEvent(button, 'click')
    .throttleTime(1000)
    .map(event => event.clientX)
    .scan((count, clientX) => count + clientX, 0)
)

autorun(() => {
    console.log("distance moved", debouncedClickDelta.current)
})
```

## ViewModel

## createViewModel

`createViewModel` takes an object with observable properties (model)
and wraps a viewmodel around it. The viewmodel proxies all enumerable properties of the original model with the following behavior:

-   as long as no new value has been assigned to the viewmodel property, the original property will be returned.
-   any future change in the model will be visible in the viewmodel as well unless the viewmodel property was dirty at the time of the attempted change.
-   once a new value has been assigned to a property of the viewmodel, that value will be returned during a read of that property in the future. However, the original model remain untouched until `submit()` is called.

The viewmodel exposes the following additional methods, besides all the enumerable properties of the model:

-   `submit()`: copies all the values of the viewmodel to the model and resets the state
-   `reset()`: resets the state of the viewmodel, abandoning all local modifications
-   `resetProperty(propName)`: resets the specified property of the viewmodel
-   `isDirty`: observable property indicating if the viewModel contains any modifications
-   `isPropertyDirty(propName)`: returns true if the specified property is dirty
-   `model`: The original model object for which this viewModel was created

You may use observable arrays, maps and objects with `createViewModel` but keep in mind to assign fresh instances of those to the viewmodel's properties, otherwise you would end up modifying the properties of the original model.
Note that if you read a non-dirty property, viewmodel only proxies the read to the model. You therefore need to assign a fresh instance not only the first time you make the assignment but also after calling `reset()` or `submit()`.

**Parameters**

-   `model` **T** 

**Examples**

```javascript
class Todo {
  \@observable title = "Test"
}

const model = new Todo()
const viewModel = createViewModel(model);

autorun(() => console.log(viewModel.model.title, ",", viewModel.title))
// prints "Test, Test"
model.title = "Get coffee"
// prints "Get coffee, Get coffee", viewModel just proxies to model
viewModel.title = "Get tea"
// prints "Get coffee, Get tea", viewModel's title is now dirty, and the local value will be printed
viewModel.submit()
// prints "Get tea, Get tea", changes submitted from the viewModel to the model, viewModel is proxying again
viewModel.title = "Get cookie"
// prints "Get tea, Get cookie" // viewModel has diverged again
viewModel.reset()
// prints "Get tea, Get tea", changes of the viewModel have been abandoned
```

## whenWithTimeout

Like normal `when`, except that this `when` will automatically dispose if the condition isn't met within a certain amount of time.

**Parameters**

-   `expr`  
-   `action`  
-   `timeout` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** maximum amount when spends waiting before giving up (optional, default `10000`)
-   `onTimeout` **any** the ontimeout handler will be called if the condition wasn't met within the given time (optional, default `()`)

**Examples**

```javascript
test("expect store to load", t => {
  const store = {
    items: [],
    loaded: false
  }
  fetchDataForStore((data) => {
    store.items = data;
    store.loaded = true;
  })
  whenWithTimeout(
    () => store.loaded
    () => t.end()
    2000,
    () => t.fail("store didn't load with 2 secs")
  )
})
```

Returns **IDisposer** disposer function that can be used to cancel the when prematurely. Neither action or onTimeout will be fired if disposed

## keepAlive

**Parameters**

-   `_1`  
-   `_2`  
-   `computedValue` **IComputedValue&lt;any>** created using the `computed` function

**Examples**

```javascript
const number = observable(3)
const doubler = computed(() => number.get() * 2)
const stop = keepAlive(doubler)
// doubler will now stay in sync reactively even when there are no further observers
stop()
// normal behavior, doubler results will be recomputed if not observed but needed, but lazily
```

Returns **IDisposer** stops this keep alive so that the computed value goes back to normal behavior

## keepAlive

MobX normally suspends any computed value that is not in use by any reaction,
and lazily re-evaluates the expression if needed outside a reaction while not in use.
`keepAlive` marks a computed value as always in use, meaning that it will always fresh, but never disposed automatically.

**Parameters**

-   `_1`  
-   `_2`  
-   `target` **[Object](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object)** an object that has a computed property, created by `@computed` or `extendObservable`
-   `property` **[string](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String)** the name of the property to keep alive

**Examples**

```javascript
const obj = observable({
  number: 3,
  doubler: function() { return this.number * 2 }
})
const stop = keepAlive(obj, "doubler")
```

Returns **IDisposer** stops this keep alive so that the computed value goes back to normal behavior

## queueProcessor

`queueProcessor` takes an observable array, observes it and calls `processor`
once for each item added to the observable array, optionally deboucing the action

**Parameters**

-   `observableArray` **[Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)&lt;T>** observable array instance to track
-   `processor`  
-   `debounce` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** optional debounce time in ms. With debounce 0 the processor will run synchronously (optional, default `0`)

**Examples**

```javascript
const pendingNotifications = observable([])
const stop = queueProcessor(pendingNotifications, msg => {
  // show Desktop notification
  new Notification(msg);
})

// usage:
pendingNotifications.push("test!")
```

Returns **IDisposer** stops the processor

## chunkProcessor

`chunkProcessor` takes an observable array, observes it and calls `processor`
once for a chunk of items added to the observable array, optionally deboucing the action.
The maximum chunk size can be limited by number.
This allows both, splitting larger into smaller chunks or (when debounced) combining smaller
chunks and/or single items into reasonable chunks of work.

**Parameters**

-   `observableArray` **[Array](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array)&lt;T>** observable array instance to track
-   `processor`  
-   `debounce` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** optional debounce time in ms. With debounce 0 the processor will run synchronously (optional, default `0`)
-   `maxChunkSize` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** optionally do not call on full array but smaller chunks. With 0 it will process the full array. (optional, default `0`)

**Examples**

```javascript
const trackedActions = observable([])
const stop = chunkProcessor(trackedActions, chunkOfMax10Items => {
  sendTrackedActionsToServer(chunkOfMax10Items);
}, 100, 10)

// usage:
trackedActions.push("scrolled")
trackedActions.push("hoveredButton")
// when both pushes happen within 100ms, there will be only one call to server
```

Returns **IDisposer** stops the processor

## now

Returns the current date time as epoch number.
The date time is read from an observable which is updated automatically after the given interval.
So basically it treats time as an observable.

The function takes an interval as parameter, which indicates how often `now()` will return a new value.
If no interval is given, it will update each second. If "frame" is specified, it will update each time a
`requestAnimationFrame` is available.

Multiple clocks with the same interval will automatically be synchronized.

Countdown example: <https://jsfiddle.net/mweststrate/na0qdmkw/>

**Parameters**

-   `interval` **([number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number) \| `"frame"`)** interval in milliseconds about how often the interval should update (optional, default `1000`)

**Examples**

```javascript
const start = Date.now()

autorun(() => {
  console.log("Seconds elapsed: ", (mobxUtils.now() - start) / 1000)
})
```

## asyncAction

_deprecated_ this functionality can now be found as `flow` in the mobx package. However, `flow` is not applicable as decorator, where `asyncAction` still is.

`asyncAction` takes a generator function and automatically wraps all parts of the process in actions. See the examples below.
`asyncAction` can be used both as decorator or to wrap functions.

-   It is important that `asyncAction should always be used with a generator function (recognizable as`function_`or`_name\` syntax)
-   Each yield statement should return a Promise. The generator function will continue as soon as the promise settles, with the settled value
-   When the generator function finishes, you can return a normal value. The `asyncAction` wrapped function will always produce a promise delivering that value.

When using the mobx devTools, an asyncAction will emit `action` events with names like:

-   `"fetchUsers - runid: 6 - init"`
-   `"fetchUsers - runid: 6 - yield 0"`
-   `"fetchUsers - runid: 6 - yield 1"`

The `runId` represents the generator instance. In other words, if `fetchUsers` is invoked multiple times concurrently, the events with the same `runid` belong toghether.
The `yield` number indicates the progress of the generator. `init` indicates spawning (it won't do anything, but you can find the original arguments of the `asyncAction` here).
`yield 0` ... `yield n` indicates the code block that is now being executed. `yield 0` is before the first `yield`, `yield 1` after the first one etc. Note that yield numbers are not determined lexically but by the runtime flow.

`asyncActions` requires `Promise` and `generators` to be available on the target environment. Polyfill `Promise` if needed. Both TypeScript and Babel can compile generator functions down to ES5.

 N.B. due to a [babel limitation](https://github.com/loganfsmyth/babel-plugin-transform-decorators-legacy/issues/26), in Babel generatos cannot be combined with decorators. See also [#70](https://github.com/mobxjs/mobx-utils/issues/70)

**Parameters**

-   `arg1`  
-   `arg2`  

**Examples**

```javascript
import {asyncAction} from "mobx-utils"

let users = []

const fetchUsers = asyncAction("fetchUsers", function* (url) {
  const start = Date.now()
  const data = yield window.fetch(url)
  users = yield data.json()
  return start - Date.now()
})

fetchUsers("http://users.com").then(time => {
  console.dir("Got users", users, "in ", time, "ms")
})
```

```javascript
import {asyncAction} from "mobx-utils"

mobx.configure({ enforceActions: true }) // don't allow state modifications outside actions

class Store {
	\@observable githubProjects = []
	\@state = "pending" // "pending" / "done" / "error"

	\@asyncAction
	*fetchProjects() { // <- note the star, this a generator function!
		this.githubProjects = []
		this.state = "pending"
		try {
			const projects = yield fetchGithubProjectsSomehow() // yield instead of await
			const filteredProjects = somePreprocessing(projects)
			// the asynchronous blocks will automatically be wrapped actions
			this.state = "done"
			this.githubProjects = filteredProjects
		} catch (error) {
			this.state = "error"
		}
	}
}
```

Returns **[Promise](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)** 

## whenAsync

_deprecated_ whenAsync is deprecated, use mobx.when without effect instead.

Like normal `when`, except that this `when` will return a promise that resolves when the expression becomes truthy

**Parameters**

-   `fn`  
-   `timeout` **[number](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number)** maximum amount of time to wait, before the promise rejects

**Examples**

```javascript
await whenAsync(() => !state.someBoolean)
```

Returns **any** Promise for when an observable eventually matches some condition. Rejects if timeout is provided and has expired

## expr

expr can be used to create temporarily views inside views.
This can be improved to improve performance if a value changes often, but usually doesn't affect the outcome of an expression.

In the following example the expression prevents that a component is rerender _each time_ the selection changes;
instead it will only rerenders when the current todo is (de)selected.

**Parameters**

-   `expr`  

**Examples**

```javascript
const Todo = observer((props) => {
    const todo = props.todo;
    const isSelected = mobxUtils.expr(() => props.viewState.selection === todo);
    return <div className={isSelected ? "todo todo-selected" : "todo"}>{todo.title}</div>
});
```

## createTransformer

Creates a function that maps an object to a view.
The mapping is memoized.

See: <https://mobx.js.org/refguide/create-transformer.html>

**Parameters**

-   `transformer`  
-   `onCleanup`  

## deepObserve

Given an object, deeply observes the given object.
It is like `observe` from mobx, but applied recursively, including all future children

Note that the given object cannot ever contain cycles and should be a tree.

As benefit: path and root will be provided in the callback, so the signature of the listener is
(change, path, root) => void

The returned disposer can be invoked to clean up the listner

**Parameters**

-   `target`  
-   `listener`  

**Examples**

```javascript
const disposer = deepObserve(target, (change, path) => {
   console.dir(change)
})
```
