# Promises

* Proposal: [HXP-NNNN](0000-promises.md)
* Author: [Aurel Bílý](https://github.com/Aurel300)

## Introduction

Add a `haxe.Promise` type to the Haxe standard library based on the A+ specification for the most part.

## Motivation

Asynchronous APIs are common in a wide variety of applications. Promises are well-suited for expressing the data flow in such APIs with a small number of idioms (`then`, `all`, etc). Several implementations of Promises exist for Haxe already<sup>[1](https://lib.haxe.org/p/thx.promise) [2](https://github.com/jdonaldson/promhx) [3](https://github.com/haxetink/tink_core)</sup>, as well as the native JavaScript Promise extern `js.Promise`.

Without relying on third-party Promise libraries, asynchronous task completion in Haxe can be facilitated with:

 - `dynamic` callback function (e.g. [`haxe.http.HttpBase.onData`](https://api.haxe.org/v/development/haxe/http/HttpBase.html#onData)) or callback function provided in an argument
 - synchronous call (the majority of methods in `sys` packages) executed in a separate thread, with some manual state management

Neither of these are particularly clean solutions. Adding a `Promise` type to the standard library would enable providing more unified APIs for all Haxe targets. `sys` classes in particular are good candidates for asynchrony, for example:

```haxe
// current API
var done = 0;
var largeData = ...;
for (i in 0...3) sys.thread.Thread.create(() -> {
    sys.io.File.saveBytes('out${i}.bin', largeData);
    done++;
  });
while (done < 3) {
  // perform other tasks ...
  Sys.sleep(.25);
}
trace("file writes done!");
```

```haxe
// API with Promises
var largeData = ...;
Promise.all([
    for (i in 0...3) sys.async.io.File.saveBytes('out${i}.bin', largeData)
  ]).then(() -> trace("file writes done!"));
// perform other tasks ...
```

Actually creating asynchronous alternatives of current stdlib APIs is out of scope of this proposal.

## Detailed design

A+ Promises are chosen as the reference implementation since it is the most widespread specification, so it should be familiar to any Javascript developer. Assuming coroutine support is added over promises, the code will also be very familiar to programmers of coroutine-supporting languages (C++, C#, Lua, Javascript/ES6).

The proposed interface:

```haxe
package haxe;

enum PromiseState {
  Pending;
  Fulfilled;
  Rejected;
}

extern class Promise<T> {
  // create a resolved promise with the given value
  public static function resolved<T>(value:T):Promise<T>;
  
  // create a rejected promise with the given error
  public static function rejected<T>(error:Dynamic):Promise<T>;
  
  // wait for all promises in an array to complete, return a promise with an
  // array of their resolved values
  public static function all<T>(promises:Array<Promise<T>>):Promise<Array<T>>;
  
  // wait for all promises in an array to complete, regardless of their type
  public static function join(promises:Array<Promise<Dynamic>>):Promise<Void>;
  
  // wait for any of the promises in the array to finish, return its value
  public static function race<T>(promises:Array<Promise<T>>):Promise<T>;
  
  public function new(
    ?resolver:(resolve:(?promise:Promise<T>, ?value:T)->Void, reject:(?error:Dynamic)->Void)->Void
  );
  
  public function done(?onFulfilled:T->Void, ?onRejected:Dynamic->Void):Void;
  
  // chain a promise, returning another promise in the given handler function
  public function then<U>(onFulfilled:T->Promise<U>, ?onRejected:Dynamic->Promise<U>):Promise<U>;
  
  // chain a promise, returning a non-promise value in the given handler function
  public function map<U>(onFulfilled:T->U, ?onRejected:Dynamic->U):Promise<U>;
  
  // catch in A+ (renamed since catch is a Haxe keyword)
  public function catchError<U>(onRejected:Dynamic->PromiseOrValue<U>):Promise<U>;
  
  public function finally<U>(onFinally:Void->U):Promise<U>;
  
  // implementation detail:
  private var state:PromiseState;
  private var value:T; // if Fulfilled
  private var error:Dynamic; // if Rejected
}
```

 - `error` may need to have its own type parameter; `catchError` may present additional typing problems (see https://github.com/HaxeFoundation/haxe/issues/8170)
 - `then` is split into `then` and `map` for when the handler function returns a `Promise` and a value, respectively

### Target specifics

On Javascript, `haxe.Promise` can simply wrap `js.lib.Promise`.

Whenever available or possible, new asynchronous APIs should use asynchronous native calls. On thread-supporting targets, synchronous APIs can be wrapped into asynchronous promises by running the synchronous call in a separate thread which will resolve the promise once done.

### Testing

Since the intended use for promises is primarily to enable the Haxe standard library to provide asynchronous APIs, Promises should be thoroughly unit-tested, e.g. using the [A+ Compliance Test Suite](https://github.com/promises-aplus/promises-tests).

## Impact on existing code

Minimal or none. `Promise` will be in the `haxe` package, which should not interfere with other promise implementations.

## Drawbacks

-

## Alternatives

With eventual support of [coroutines in Haxe](https://github.com/RealyUniqueName/Coro), API authors and the Haxe standard library could provide asynchronous API functions as coroutines instead of returning Promises. Since coroutines can easily be created from promises, this should not be an issue – promises and coroutines can coexist in Haxe.

## Opening possibilities

As shown by the example in the [motivation](#motivation) section, Promises in the standard library will allow providing asynchronous alternatives to various `sys` APIs, on par with standard libraries such as the one of node.js.

Library authors and programmers in general will also benefit from being able to easily create asynchronous APIs.

## Unresolved questions

-
