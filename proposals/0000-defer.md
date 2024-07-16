# Add Defer

* Proposal: [HXP-0000](0000-defer.md)
* Author: [Dawson Davis](https://github.com/DawDavis)

## Introduction

`defer` is a keyword lifted directly from GoLang, which if implemented would allow for the automatic execution of an expression upon the exit of the current scope.

## Motivation

Currently, there is no direct way to call code no matter the execution path of the following control-flow. This is unfortunate, as it easily leads to the duplication of expressions, or a rapidly nested series of calls to try...catch. If added, `defer` would allow for a clean single point of implementation for various cleanup and house-keeping  tasks.

In my own code, multithreading is a source of much pain - and a unified control flow would allow me to make changes faster without ctrl-c, ctrl-v against several possible outcomes. In GoLang, it is the unified `defer` that handles recovery from panics, file handle cleanup, IO closure, and much more. 

As a sane limit to the `defer` mentioned here, we should presume that the deferred operation only passes arguments by reference and for all intents and purposes is surrounded in an anonymizing function.

## Detailed design

The defer keyword would capture an expression inside a `()->Void` function and then delay the execution of the deferred expression up until the current function scope is exited either by return or error. This is analogous to the function a `finally` block has, however the `defer` keyword is more flexible, and it prevents nesting of try...catch blocks, which lead to overly indented and hard to read code.

Additionally, `defer` should operate on any expression, requiring no special implementation annotations or interfaces. This is so that any library can take advantage of deferred statements out of the box. This also allows the possibility of deferring anonymous function calls, which can handle multiple operations in a single `defer` call.

In the below examples, I will show a macro-alike implementation in pure haxe, which should be compatible with all targets. Target specific implementations are of course possible, however to maintain compatibility I have simply used the most widespread implementation possible.

Example code, as it stands in 4.3.X:

```haxe
import sys.thread.Mutex;

class SomeThreadsafeObject {
    var mut = new Mutex();

    public function exampleOperation() {
        mut.acquire();

        try {
            someFallibleOperation();
        } catch (e) {
            //release on the sad path
            mut.release();
            throw e;
        }
        //release on the happy path
        mut.release();
    }
}
```

With the proposed change, the above code could be simplified into the following:

```haxe
import sys.thread.Mutex;

class SomeThreadsafeObject {
    var mut = new Mutex();

    public function exampleOperation() {
        mut.acquire();
        defer mut.release();

        someFallibleOperation();
    }
}
```

Under the hood this change could be described using the following modification of the source code, as shown below:

```haxe
import sys.thread.Mutex;

class SomeThreadsafeObject {
    var mut = new Mutex();

    public function exampleOperation() {
        mut.acquire();
        var deferred0000:() -> Void = () -> { mut.release(); };
        {
            try {
                someFallibleOperation();
            } catch (e) {
                deferred0000();
                throw e;
            }
            deferred0000();
        }
    }
}
```

This change should also allow post-return cleanup:

```haxe
public function exampleOperation():Bool {
    mut.acquire();
    defer mut.release();
    return true;
}
```

Which would then anonymize into the following:

```haxe
public function exampleOperation():Bool {
    mut.acquire();
    var deferred0000:() -> Void = () -> { mut.release(); };
    var deferredReturn0000:Bool;
    try {
        deferredReturn0000:()->Bool = () -> {
            return true;
        }
    }
    return true;
}
```

This implementation using anonymized function bodies however is not optimal because it could obfuscate the original caller location - if done in the compiler, debugging information should obviously reflect the original pre-expansion code.

To abstract away the implementation details, we can also view this as the abstract pseudo code template:

```haxe
function functionName(...):ReturnType {

    exprs();

    defer exprToDefer(args...);

    body();
    return x;
}
```

which would then be expanded to:

```haxe
function functionName(...):ReturnType {

    exprs();

    var deferredExpr:()->Void = ()->{ exprToDefer(args...); }
    var deferredReturnValue:ReturnType;
    try {
        deferredReturnValueFunc:()->ReturnType = ()->{
            body();
            return x;
        };
        deferredReturnValue = deferredreturnValueFunc();
    } catch (e) {
        deferredExpr();
        throw e;
    }
    deferredExpr();
    return deferredReturnValue;
}
```

## Impact on existing code

The impact of the addition of a `defer` keyword should be minimal, as it is a relatively uncommon word in most haxe code. The migration to using defer should be relatively straightforward, as it will still be possible to use other less automated cleanup methods.

## Drawbacks

The drawbacks are simply put: memory and speed. The memory footprint of the code will increase on some targets substantially during runtime, especially if static analysis doesn't do a sanity pass to remove redundancies that the defer statements may add.

The other issue is one of speed. On some platforms where try...catch is not as optimized, we may see slowdowns during multiple defers as the target deals with the nested catch statements. In most use-cases defer is only used a few times per call, and the safety that comes with proper defer use probably outweighs any impact in speed.

Someone affected by the change could also just revert their code to an older style of flow-control, and avoid the issues altogether.

## Alternatives

This entire proposal hinges on the idea that `finally`will not be introduced. I acknowledge that this is a bodge that sits loosely on the shoulders of try...catch. However due to the flexibility of defer, it would fit the niche nicely.

Additionally the changes in [Auto Closing #119](https://github.com/HaxeFoundation/haxe-evolution/pull/119) would probably negate the need for a `defer` keyword, however those changes are limited to a `closeable` interface as the proposal currently stands. The `defer` keyword would be more flexible, which is why I decided to make a separate implementation request.

## Opening possibilities

This change is mostly self-contained, however it will make writing standard libraries easier and less error-prone, as there will be a unified way to close files, pipes, streams, etc. once it is implemented.

## Unresolved questions

* What stage of parsing the implementation would reside in?

* How the exact variables are captured during compilation?

* the `defer` statement could throw an exception. This is troublesome, as it could interrupt the control-flow. In GoLang, errors are not propagated as throws, so the implementation doesn't worry about such events. We could add an `ignore( x )`macro to wrap the defer in a try...catch
