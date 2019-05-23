# New `sys` APIs

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Aurel Bílý](https://github.com/Aurel300)

## Introduction

Improved API for both synchronous and asynchronous filesystem operations based on Node.js; improved networking API; asynchrony primitives; I/O streams.

## Motivation

### Asynchrony

There is currently no good way to asynchronously perform many `sys`-related tasks (without manually creating `Thread`s). Two basic primitives are added to the library:

 - [events](#events) (and listeners)
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

Non-blocking socket operations are inconvenient to use in the current API even though they are the only (non-`Thread`) solution to some real-time network communication problems. IPC communication is not possible, UDP sockets are not fully featured.

There is a lack of proper unit testing of the networking APIs. Certain platforms also miss full implementations of various parts of the networking API. (See https://github.com/HaxeFoundation/haxe/issues/6933, https://github.com/HaxeFoundation/haxe/issues/6816)

## Detailed design

Modified modules (new API + backward compatibility):

 - [`haxe.io.Path`](haxe/io/Path.hx)
 - [`sys.FileStat`](sys/FileStat.hx)
 - [`sys.FileSystem`](sys/FileSystem.hx)
 - [`sys.io.File`](sys/io/File.hx)
 - [`sys.net.UdpSocket`](sys/net/UdpSocket.hx)

Added modules:

 - [`haxe.Error`](haxe/Error.hx) - for reporting errors, see [errors](#errors)
 - [`haxe.ErrorType`](haxe/ErrorType.hx)
 - [`haxe.NoData`](haxe/NoData.hx) - type to represent an absence of data in generics (e.g. `Callback<NoData>`)
 - [`haxe.async.Callback`](haxe/async/Callback.hx) - generic type to represent an error-first callback, see [callbacks](#callbacks)
 - [`haxe.async.Event`](haxe/async/Event.hx) - see [events](#events)
 - [`haxe.async.Listener`](haxe/async/Listener.hx) - event listener
 - [`haxe.io.Duplex`](haxe/io/Duplex.hx) - see [streams](#streams)
 - [`haxe.io.FilePath`](haxe/io/FilePath.hx) - see [non-unicode filepaths](#non-unicode-filepaths)
 - [`haxe.io.IReadable`](haxe/io/IReadable.hx)
 - [`haxe.io.IStream`](haxe/io/IStream.hx)
 - [`haxe.io.IWritable`](haxe/io/IWritable.hx)
 - [`haxe.io.Readable`](haxe/io/Readable.hx)
 - [`haxe.io.Stream`](haxe/io/Stream.hx)
 - [`haxe.io.Writable`](haxe/io/Writable.hx)
 - [`sys.DirectoryEntry`](sys/DirectoryEntry.hx)
 - [`sys.FileAccessMode`](sys/FileAccessMode.hx)
 - [`sys.FileCopyFlags`](sys/FileCopyFlags.hx)
 - [`sys.FileMode`](sys/FileMode.hx)
 - [`sys.FileOpenFlags`](sys/FileOpenFlags.hx)
 - [`sys.FileWatcher`](sys/FileWatcher.hx)
 - [`sys.async.FileSystem`](sys/async/FileSystem.hx)
 - [`sys.async.Http`](sys/async/Http.hx)
 - [`sys.async.net.Socket`](sys/net/Socket.hx)
 - [`sys.io.AsyncFile`](sys/io/AsyncFile.hx)
 - [`sys.io.FileReadStream`](sys/io/FileReadStream.hx)
 - [`sys.io.FileWriteStream`](sys/io/FileWriteStream.hx)
 - [`sys.net.Dns`](sys/net/Dns.hx)
 - [`sys.net.Net`](sys/net/Net.hx)
 - [`sys.net.Server`](sys/net/Server.hx)
 - [`sys.net.UdpSocket`](sys/net/UdpSocket.hx)
 - [`sys.net.Url`](sys/net/Url.hx)

Relevant Node.js APIs:

 - [`dgram`](https://nodejs.org/api/dgram.html)
 - [`dns`](https://nodejs.org/api/dns.html)
 - [`fs`](https://nodejs.org/api/fs.html)
 - [`http`](https://nodejs.org/api/http.html)
 - [`net`](https://nodejs.org/api/net.html)
 - [`path`](https://nodejs.org/api/path.html)
 - [`stream`](https://nodejs.org/api/stream.html)
 - [`url`](https://nodejs.org/api/url.html)

See also the [detailed differences from Node.js APIs](NODE-DIFF.md) and [Haxe 4 breaking chagnes](BREAKING.md).

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

### Events

A type-safe system for emitting events, similar to `tink_core` `Signal`s is added. An `Event<T>` is simply an abstract over an array of listeners (`Listener<T>`). An event-emitting object has a number of `final` events.

```haxe
class Example {
  public final eventFoo = new Event<NoData>();
  public final eventBar = new Event<String>();
  public function new() super();
  public function emitEvents() {
    eventFoo.emit(new NoData());
    eventBar.emit("hello");
  }
}

class Main {
  static function main():Void {
    var example = new Example();
    example.eventFoo.on(() -> trace("event foo"));
    example.eventBar.on(str -> trace("event bar", str));
    example.emitEvents();
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

The Node.js API has a concept of file descriptors, represented by a single integer. To avoid issues with platforms without explicit file descriptor numbers, `sys.io.File` is an `abstract`, similar to the new threading API.

Various `fs.f*` methods from Node.js which take `fd` as their first argument are moved into their own methods in the `File` abstract.

### Synchronous / asynchronous versions

To avoid the `someMethod` + `someMethodSync` naming scheme present in Node.js, the two versions are more clearly split:

 - `sys.FileSystem` and `sys.async.FileSystem` (static methods)
 - `sys.io.File` has an `async` field for asynchronous instance methods

```haxe
// synchronously:
var file = sys.FileSystem.open("file.txt", Read);
var data = file.readFile();

// asynchronously:
sys.async.FileSystem.open("file.txt", Read, (err, file) -> {
    file.readFile((err, data) -> {
        // ...
      });
  });
```

### Non-Unicode filepaths

In Node.js, wherever a path is expected as an argument, a `Buffer` can be provided, equivalent to `haxe.io.Bytes`. Similarly, whenever paths are to be returned, either a `String` or a `Buffer` is returned, depending on the `encoding` option (`"utf8"` or `"buffer"`).

It would be awkward to mirror this behaviour in Haxe, so instead, the assumption is made that filepaths will be Unicode most of the time, and `haxe.io.FilePath` (an `abstract` over `String`) is used consistently in the new API. In the rare cases that non-Unicode paths are returned, they are escaped into a Unicode string. The original `Bytes` can be obtained with `FilePath.decode(path)`. There is also the inverse `FilePath.encode(bytes)`.

See https://github.com/HaxeFoundation/haxe/issues/8134

### Backward compatibility

The methods in the current `sys.FileSystem` and `sys.io.File` APIs will be kept for the time being, as `inline`s using the new methods. The names of the methods in Node.js are arguably less intuitive (e.g. `mkdir` instead of `createDirectory`), but they were kept to retain familiarity.

### Target specifics

Where possible, the asynchronous methods should use native calls. For some targets this might not be possible, so in the worst-case scenario these methods will run the synchronous call in a `Thread`, then trigger the callback once done.

For many targets, wrapping libuv (the library that powers Node.js APIs) will be the most straight-forward implementation option.

(**TODO:** research individual APIs on remaining targets)

 - cpp
 - cs
 - eval - [libuv bindings for OCaml](https://github.com/fdopen/uwt) ?
 - hl - libuv bindings already started
 - java, jvm
 - js (with `hxnodejs`) - mostly trivial mapping since it is the Node.js API
 - lua - [luvit](https://github.com/luvit/luvit)
 - neko
 - php
 - python

### Testing

The majority of tests for the current `sys` classes should be reused. It may be worthwhile to adapt the existing tests to test both implementations (with a forced synchronous operation on `sys.async`) so tests are not duplicated. Additional tests should be written to test async-specific features, such as writing multiple files in parallel.

For methods that were not present in the original APIs, some tests may be based on the extensive [Node.js test suite](https://github.com/nodejs/node/tree/master/test/parallel).

## Impact on existing code

Existing code should not be affected:

 - completely new APIs will be in new packages
 - new APIs which are largely compatible with the old APIs still keep the methods of the old APIs for backward compatibility

## Drawbacks

-

## Alternatives

-

## Opening possibilities

 - better haxelib

## Unresolved questions

To be determined before implementation (in PR discussions):

 - [error reporting style](#errors)
 - specifics of packages, class names generally
 - currently all filesize and file position arguments are `Int`, but this only allows sizes of up to 2 GiB
   - should we use `haxe.Int64`?
   - is the support of `haxe.Int64` good enough on sys targets
   - Node.js uses the `Number` type, which has at least 53 bits of integer precision
 - Haxe compatibility - Dns (Host), Socket
 - Https - mostly a copy of the Http APIs, some extra SSL-specific options