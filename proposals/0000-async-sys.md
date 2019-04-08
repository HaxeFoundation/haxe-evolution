# Promise-based asynchronous `sys` API

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Aurel Bílý](https://github.com/Aurel300)

## Introduction

Add A+-like Promises. Add a `sys.async` package with Promise-based asynchronous alternatives to some `sys` methods.

## Motivation

For any larger project it may be necessary to perform other tasks while a system call is being processed in the background. Libraries in many Haxe targets include system-related methods which may not return/produce a result immediately:

 - [C++ - `boost.asio`](http://think-async.com/Asio/boost_asio_1_12_2/doc/html/boost_asio.html) (or just [`asio`](http://think-async.com/Asio/)) - with `co_async/co_await` coroutines
 - [C#](https://docs.microsoft.com/en-us/dotnet/standard/io/asynchronous-file-i-o) - with `async/await` coroutines
 - [node.js - `fs`](https://nodejs.org/api/fs.html) - methods take a `callback` argument, although in practice [`promisify`](https://nodejs.org/api/util.html#util_util_promisify_original) is often used to transform callback-taking methods into Promise-returning methods
 - [Java - `AsynchronousFileChannel`](https://docs.oracle.com/javase/7/docs/api/java/nio/channels/AsynchronousFileChannel.html) - methods return a `Future` (largely identical to Promise from an API perspective)
 - [Python - Twisted](https://twistedmatrix.com/trac/) - networking methods
 - [lua - LuaNode](https://github.com/ignacio/luanode) (might be a dead project) - node.js-like I/O

Importantly, these asynchronous APIs **notify** the caller code on operation completion or failure ([Proactor design pattern](https://en.wikipedia.org/wiki/Proactor_pattern)). This is superior to non-blocking operations, which may perform a small part of an operation, e.g. writing `n` bytes to a file, then returning to the caller code immediately. With such APIs the programmer is responsible for writing a wrapper around the API to make sure the task is finished completely. This is error-prone and imitating a true async API interface most likely involves threading.

As an example, we might want to write a couple of very large files in parallel. With the current libraries:

```haxe
class Example {
  public static function main():Void {
    var done = 0;
    var data = haxe.io.Bytes.alloc(1024 * 1024 * 700);
    for (i in 0...3) sys.thread.Thread.create(() -> {
        sys.io.File.saveBytes('out${i}.bin', data);
        done++;
      });
    while (done < 3) {
      Sys.sleep(.25);
    }
    trace("file writes done!");
  }
}
```

Note the use of thread, an explicit loop, and manual state control. Now, assuming we have a `sys.async.io.File.saveBytes()` method returning an A+ `Promise` (see [detailed design](#detailed-design)):

```haxe
class Example {
  public static function main():Void {
    var data = haxe.io.Bytes.alloc(1024 * 1024 * 700);
    Promise.all([
        for (i in 0...3) sys.async.io.File.saveBytes('out${i}.bin', data)
      ]).then(() -> trace("file writes done!"));
  }
}
```

A+ Promises are chosen as the reference implementation since it is the most widespread specification, so it should be familiar to any Javascript developer. Assuming coroutine support is added over promises, the code will also be very familiar to programmers of coroutine-supporting languages (C++, C#, Lua, Javascript/ES6).

## Detailed design

The following methods will get their async counterparts:

 - `sys.db.Mysql` (static)
   - `connect(params:...):Promise<Connection>`
 - `sys.db.Sqlite` (static)
   - `open(file:String):Promise<Connection>`
 - `sys.io.File` (static)
   - `copy(srcPath:String, dstPath:String):Promise<Void>`
   - `getBytes(path:String):Promise<Bytes>`
   - `getContent(path:String):Promise<String>`
   - `saveBytes(path:String, bytes:Bytes):Promise<Void>`
   - `saveContent(path:String, content:String):Promise<Void>`
 - `sys.io.Process` (static)
   - `open(cmd:String, ?args:Array<String>, ?detached:Bool):Promise<Process>` (counterpart to the constructor)
 - `sys.io.Process` (instance)
   - `close():Promise<Void>`
   - `exitCode():Promise<Int>`
   - `kill():Promise<Void>`
 - `sys.net.Host` (instance)
   - `reverse():Promise<String>`
 - `sys.net.Socket` (static)
   - `select(read, write, others):Promise<{read, write, others}>`
 - `sys.net.Socket` (instance)
   - `accept():Promise<Socket>`
   - `connect(host:Host, port:Int):Promise<Void>`
 - `sys.ssl.Certificate` (static)
   - `loadDefaults():Promise<Certificate>`
   - `loadFile(file:String):Promise<Certificate>`
   - `loadPath(path:String):Promise<Certificate>`
 - `sys.ssl.Key` (static)
   - `loadFile(file:String, ?isPublic:Bool, ?pass:String):Promise<Key>`
 - `sys.ssl.Socket` (instance)
   - `handshake():Promise<Void>`
 - `sys.thread.Deque` (instance)
   - `pop():Promise<T>` (this could just be added to the existing `Deque`)
 - `sys.thread.Lock` (instance)
   - `wait():Promise<Void>` (this could just be added to the existing `Lock`)
 - `sys.FileSystem` (static)
   - `createDirectory(path:String):Promise<Void>`
   - `deleteDirectory(path:String):Promise<Void>`
   - `deleteFile(path:String):Promise<Void>`
   - `exists(path:String):Promise<Bool>`
   - `isDirectory(path:String):Promise<Bool>`
   - `readDirectory(path:String):Promise<Array<String>>`
   - `rename(path:String, newPath:String):Promise<Void>`
   - `stat(path:String):Promise<sys.FileStat>`
 - `sys.Http` (static)
   - `requestUrl(url:String):Promise<String>`

A `Promise` class will be added to the standard library, either in the top-level or the `haxe` package. Its API will mimic the A+ API where possible (A+ is the `Promise` API used in Javascript). The differences are highlighted:

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
  // resolve in A+
  public static function resolved<T>(with:T):Promise<T>;
  
  // reject in A+
  public static function rejected<T>(with:Dynamic):Promise<T>;
  
  public static function all<T>(promises:Array<PromiseOrValue<T>>):Promise<Array<T>>;
  
  public static function race<T>(promises:Array<PromiseOrValue<T>>):Promise<T>;
  
  // bluebird specific
  // also already in haxe.Timer
  public static function delay(millis:Float):Promise<Void>;
  
  public var state(default, null):PromiseState;
  
  // if Fulfilled
  public var value(default, null):T;
  
  // if Rejected
  public var error(default, null):Dynamic;
  
  public function new(?resolver:(PromiseOrValue<T>->Void)->(Dynamic->Void)->Void);
  
  public function done(?onFulfilled:T->Void, ?onRejected:Dynamic->Void):Void;
  
  public function then<U>(onFulfilled:T->PromiseOrValue<U>, ?onRejected:Dynamic->PromiseOrValue<U>):Promise<U>;
  
  // catch in A+
  public function catchError<U>(onRejected:Dynamic->PromiseOrValue<U>):Promise<U>;
  
  public function finally<U>(onFinally:Void->U):Promise<U>;
}
```

### Target specifics

Where possible, the `sys.async` methods should use native asynchronous methods (see example links in the [motivation](#motivation) section). For some targets this might not be possible, so in the worst-case scenario these methods will wrap a `Thread` with a Promise API.

### Testing

Since the `sys.async` package will be built on top of the Promise implementation, Promises should be thoroughly unit-tested, e.g. using the [A+ Compliance Test Suite](https://github.com/promises-aplus/promises-tests).

The majority of tests for `sys` classes should be reused. It may be worthwhile to adapt the existing tests to test both implementations (with a forced synchronous operation on `sys.async`) so tests are not duplicated. Additional tests should be written to test async-specific features, such as writing multiple files in parallel.

## Impact on existing code

Existing code should not be affected, since the new classes will be in the new `sys.async` package and the old `sys` API will not be changed.

The optional migration from `sys` to `sys.async` where appropriate amounts to a possibly large refactor.

## Drawbacks

## Alternatives

Alternatives to returning `Promise`s:

 - callback arguments on the methods (as in node.js) - difficult to chain, leads to deeply-nested functions (callback hell)
 - object instance per asynchronous operation with `dynamic` functions (as in `sys.Http`) - cannot chain, a lot of additional classes
 - skip `Promise`s completely and use [coroutines](https://github.com/RealyUniqueName/Coro) - the Coro API is already based on A+ Promises

Alternatives to having asynchronous methods altogether:

 - using manual threading (see example in [motivation](#motivation)) - more error-prone

## Opening possibilities

## Unresolved questions

 - should this proposal be split into two? "Promises" and "Async `sys` API"
 - drawbacks, opening possibilities?

### TODO before PR

 - [ ] finalise the list of async methods
 - [ ] agree on the `Promise` API
