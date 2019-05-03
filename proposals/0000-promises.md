# Promises

* Proposal: [HXP-NNNN](0000-promises.md)
* Author: [Aurel Bílý](https://github.com/Aurel300)

## Introduction

Add a `Promise` type to the Haxe standard library based on the A+ specification for the most part.

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
enum PromiseOrValue<T> {
  Promise(_:Promise<T>);
  Value(_:T);
  
  // may be necessary to avoid errors (Void cannot be used as a value)
  VoidValue;
}

enum PromiseState {
  Pending;
  Fulfilled;
  Rejected;
}

extern class Promise<T> {
  public static function resolved<T>(with:T):Promise<T>;
  
  public static function rejected<T>(with:Dynamic):Promise<T>;
  
  public static function all<T>(promises:Array<PromiseOrValue<T>>):Promise<Array<T>>;
  
  public static function race<T>(promises:Array<PromiseOrValue<T>>):Promise<T>;
  
  public var state(default, null):PromiseState;
  
  // if Fulfilled
  public var value(default, null):T;
  
  // if Rejected
  public var error(default, null):Dynamic;
  
  public function new(?resolver:(PromiseOrValue<T>->Void)->(Dynamic->Void)->Void);
  
  public function done(?onFulfilled:T->Void, ?onRejected:Dynamic->Void):Void;
  
  public function then<U>(onFulfilled:T->PromiseOrValue<U>, ?onRejected:Dynamic->PromiseOrValue<U>):Promise<U>;
  
  // catch in A+ (renamed since catch is a Haxe keyword)
  public function catchError<U>(onRejected:Dynamic->PromiseOrValue<U>):Promise<U>;
  
  public function finally<U>(onFinally:Void->U):Promise<U>;
}
```

`error` may need to have its own type parameter; `catchError` may present additional typing problems (see https://github.com/HaxeFoundation/haxe/issues/8170).

### Target specifics

On Javascript, `Promise` can simply wrap `js.lib.Promise`.

Whenever available or possible, new asynchronous APIs should use asynchronous native calls. On thread-supporting targets, synchronous APIs can be wrapped into asynchronous promises by running the synchronous call in a separate thread which will resolve the promise once done.

## Impact on existing code

Minimal; `Promise` will be in the top-level package, which may interfere with projects using other Promise implementations.

## Drawbacks

-

## Alternatives

 - coroutines

## Opening possibilities

As shown by the example in the [motivation](#motivation) section, Promises in the standard library will allow providing asynchronous alternatives to various `sys` APIs, on par with standard libraries such as the one of node.js.

Library authors and programmers in general will also benefit from being able to easily create asynchronous APIs.

## Unresolved questions

-
