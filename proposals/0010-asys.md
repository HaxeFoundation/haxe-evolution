# New `asys` APIs

* Proposal: [HXP-0010](0010-asys.md)
* Author: [Aurel Bílý](https://github.com/Aurel300)
* Status: [to be implemented](https://github.com/Aurel300/haxe-sys)

## Introduction

Improved API for both synchronous and asynchronous filesystem operations; improved networking API; improved threading and process API; asynchrony primitives; I/O streams.

## Motivation

### Asynchrony

There is currently no good way to asynchronously perform many `sys`-related tasks (without manually creating `Thread`s). Two basic primitives are added to the library:

 - [signals](#signals) (and listeners)
 - [unified callback style](#callbacks)

### Streams

The current Haxe API contains `haxe.io.Input` and `haxe.io.Output` for input and output streams. These lack:

 - ability to express a read **and** write stream (`sys.io.File` has two separate streams rather than one RW stream)
 - pipelining without manual chunking
 - proper asynchronous operations
 - automatically pacing streams with different data emission / consumption rates

### Filesystem

The current filesystem APIs in Haxe lack a number of important features:

 - asynchronous tasks
 - changing permissions, owners of files
 - symlink operations
 - watching for changes

### Networking

Non-blocking socket operations are inconvenient to use in the current API even though they are the only (non-`Thread`) solution to some real-time network communication problems. IPC communication is not possible, UDP sockets are not fully featured, DNS lookup is always synchronous and not fully featured.

There is a lack of proper unit testing of the networking APIs. Certain platforms also miss full implementations of various parts of the networking API. (See https://github.com/HaxeFoundation/haxe/issues/6933, https://github.com/HaxeFoundation/haxe/issues/6816)

### Threads, processes, and timers

Some Haxe targets (e.g. eval) have problematic implementations of threads which can result in unexpected deadlocks or crashes. It is not possible to pass handles (sockets or open files) to open processes (IPC); there is no standardised message passing for child processes.

## Detailed design

The APIs will be implemented as direct wrappers of [libuv](http://libuv.org/) (which is the foundation of Node.js APIs) on targets which allow this, i.e. eval, Neko, HashLink, hxcpp, and Lua. The hxnodejs library will be updated to map Node.js APIs to the new `sys` APIs.

Java, C#, PHP, and Python may at first expose the new `sys` APIs by requiring a native library (`dll`, `so`, `dylib`). Proper target-native APIs can be added over time, particularly after an in-depth test suite is available.

The full implementation status is available in the [haxe-sys](https://github.com/Aurel300/haxe-sys) repository.

### Errors

A `haxe.Error` class is added to unify error reporting in the system APIs. It has a `message` field which contains the human-readable description of the error. It also includes a `type` field which can be `switch`-ed on.

```haxe
try {
  sys.FileSystem.someOperation();
} catch (err:haxe.Error) {
  trace("error!", err);
}
// or
try {
  sys.FileSystem.someOperation();
} catch (err:haxe.Error) {
  switch (err.type) {
    case FileNotFound: // it's fine
    case _: throw err;
  }
}
```

> **Unresolved question:**
> 
> There are multiple ways of expressing proper type-safe errors for the filesystem API:
> - errors represented by a single `enum` (`sys.FileSystemError`), with the individual cases containing all the information of that particular error
>   - awkward to catch individual errors (any `catch` would need a `switch`)
>   - fewer classes to maintain, less work to throw errors (the case names the error, so no message is needed)
> - errors represented by sub-classes of a single base class
>   - possible to catch individual subclasses in separate `catch` blocks
>   - many classes in the package (could be moved into a sub-package for errors?)
> - base class `Error` + enum for types, as implemented in the draft now
> 
> The primary aim for any solution is to be able to catch specific types of errors without having to rely on string comparison.

### Signals

A type-safe system for emitting signals (similar to events) is added, similar to `tink_core`. A `Signal<T>` is simply an abstract over an array of listeners (`Listener<T>`). A signal-emitting object has a number of `final` signal instances.

```haxe
class Example {
  public final fooSignal = new Signal<NoData>();
  public final barSignal = new Signal<String>();
  public function new() {}
  public function emit() {
    fooSignal.emit(new NoData());
    barSignal.emit("hello");
  }
}

class Main {
  static function main():Void {
    var example = new Example();
    example.fooSignal.on(() -> trace("signal foo"));
    example.barSignal.on(str -> trace("signal bar", str));
    example.emit();
  }
}
```

Currently no efforts were made to "hide" the `emit` method (like the `Signal` and `SignalTrigger` distinction made in `tink_core`).

### Callbacks

Asynchronous methods are identical to their synchronous counter-parts, except:

 - their return type is `Void`
 - they have an additional, required `callback` argument of type `Callback<DataType>` or `Callback<NoData>`
   - first argument passed to the callback is a `haxe.Error`, or `null` if no error occurred
   - any additional arguments represent the data returned by the call, analogous to the return type of the synchronous method; if the synchronous method has a `Void` return type, the callback takes no additional arguments
   - `Callback<T>` is an abstract which has some `from` methods, allowing a callback to be created from functions with a simpler signature (e.g. a `Callback<NoData>` from `(err:Error)->Void`)

### Flags, modes, constants

Several methods in the API accept constants or a combination of flags. Constants (where the argument is *exactly one of* a set of options) have been converted to an `enum` or `enum abstract`. Flags (where the argument is *zero or more of* a set of options) have been converted to an `abstract` over `Int`, with an overloaded `|` operator.

### Streams

At the core of a lot of Node.js APIs lie [streams](https://nodejs.org/api/stream.html), which are abstractions for data consumers (`Writable`), data producers (`Readable`), or a mix of both (`Duplex` or `Transform`). Streams enable better composition of data operations with methods such as `pipeline`. There is also a mechanism to minimise buffering of data in memory (`highWaterMark`, `drain`) when combining streams.

### File descriptors

The libuv API has a concept of file descriptors, represented by a single integer. To avoid issues with platforms without explicit file descriptor numbers, `sys.io.File` is an `abstract`, similar to the new threading API.

Various methods which take a file descriptor as their first argument are moved into their own methods in the `File` abstract.

### Synchronous / asynchronous versions

To avoid the `someMethod` + `someMethodSync` naming scheme present in Node.js, the two versions are more clearly split:

 - `asys.FileSystem` and `asys.AsyncFileSystem` (static methods)
 - `asys.io.File` and `asys.io.AsyncFile` (instance methods)

`asys.io.File` exposes an `async` field to access the `asys.io.AsyncFile` corresponding to a particular file.

```haxe
// synchronously:
var file = asys.FileSystem.open("file.txt", Read);
var data = file.readFile();

// asynchronously:
asys.AsyncFileSystem.open("file.txt", Read, (err, file) -> {
    file.async.readFile((err, data) -> {
        // ...
      });
  });
```

### Non-Unicode filepaths

In libuv, wherever a path is expected as an argument, a `char *` can be provided, equivalent to `haxe.io.Bytes`. Similarly, whenever paths are to be returned, either a `char *` is returned.

It would be awkward to require `Bytes` objects as file paths in Haxe, so instead, the assumption is made that filepaths will be valid Unicode most of the time, and `haxe.io.FilePath` (an `abstract` over `String`) is used consistently in the new API. In the rare cases that non-Unicode paths are returned, they are escaped into a Unicode string. The original `Bytes` can be obtained with `FilePath.decode(path)`. There is also the inverse `FilePath.encode(bytes)`.

See https://github.com/HaxeFoundation/haxe/issues/8134

### Backward compatibility

The new APIs reserved for system targets will be available in a new top-level package `asys`. Some cross-platform types will be added to the `haxe` package. A `sys-compat` library will be provided to map the old `sys` APIs to the new `asys` package for easier transitioning and testing, although the old `sys` APIs will remain untouched when the library is not used.

### Testing

The majority of tests for the current `sys` classes should be adapted and reused. It may be worthwhile to adapt the existing tests to test both implementations (with a forced synchronous operation on `sys.async`) so tests are not duplicated. Additional tests should be written to test async-specific features, such as writing multiple files in parallel.

For methods that were not present in the original APIs, some tests may be based on the extensive [libuv test suite](https://github.com/libuv/libuv/tree/v1.x/test) or the [Node.js test suite](https://github.com/nodejs/node/tree/master/test/parallel).

## Impact on existing code

Existing code should not be affected, unless it uses an `asys` package.

## Drawbacks

Wrapping libuv allows easily supporting new APIs without several separate implementations. This approach may reduce portability on some of our targets, see [detailed design](#detailed-design).

## Alternatives

There are currently no alternatives in Haxe libraries with a similar feature range. It might be possible on some of Haxe targets to back the new APIs with target-native features, but it would also seriously increase the complexity of this project.

## Opening possibilities

 - better haxelib
 - libuv available in the OCaml code of the compiler - threading and parallelisation may be possible

## Unresolved questions

 - [error reporting style](#errors)
 - currently all filesize and file position arguments are `Int`, but this only allows sizes of up to 2 GiB
   - use `haxe.Int64`? (dependent on better support on all sys targets, e.g. HashLink)
