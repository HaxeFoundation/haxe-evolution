# Auto Closing

* Proposal: [HXP-NNNN](NNNN-auto-closing.md)
* Author: [Aidan Lee](https://github.com/Aidan63)

## Introduction

Many classes contain a `close` function responsible for cleaning up handles, native resources, or other things which are outside of the control of the GC or which may not want to be kept around until the GC runs. This proposal introduces a new `autoclose` keyword to variable declarations which ensures the `close` function is automatically called when execution of the current block ends.

## Motivation

Leaking resources from these classes is very easy as the user has to manually call `close` and needs to take precautions against exceptions which could prevent the function being executed. Haxe also lacks a `finally` for exceptions which has been the traditional recommendation in other languages which means users have to write excessively verbose try catches to deal with any exceptions and not leak resources.

```haxe
function bar() {
    final reader = File.read("input.txt", false);
    try
    {
        final writer = File.write("output.txt", false);

        try
        {
            while (!reader.eof()) {
                writer.writeString(reader.readLine());
            }
        }
        catch (exn)
        {
            writer.close();

            throw exn;
        }

        writer.close();
    }
    catch (exn) {
        reader.close();

        throw exn;
    }

    reader.close();
}
```

You could change the above slightly to a single try catch but you then need to do null checks against each class with a close function to check that it was constructed, so neither option is great. I can't speak for everyone but I certaintly don't write code like this so I suspect there's a lot of potentially leaky haxe code out there for failure paths. 

The `autoclose` keyword solves this problem for the user by ensuring constructed classes with a `close` function have it executed at the end of the code block or if an exception is thrown.

```haxe
autoclose final reader = File.read("input.txt", false);
autoclose final writer = File.write("output.txt", false);

while (!reader.eof()) {
    writer.writeLine(reader.readLine());
}
```

## Detailed design

The `autoclose` keyword can be prefixed to any variable delaration expression where the type contains a public `close` function with a `Void->Void` signature. This allows the keyword to be easily used with all existing closable standard library types and 3rd party libraries without modifications requiring some interface to be implemented.

The try catch nesting as shown in the examples in the previous section will be generated for the user to ensure close is called in normal and exceptional cases.

Null checks should be added before `close` is called and in the case of a null variable no `close` call should be made and no exception thrown. Variables of `Dynamic` should not be allow to use the `autoclose` keyword.

Coroutines support can be added by having `autoclose` also be allowed on variable of types which have a `close` coroutine function of type `Void->Unit`. If a class has both a `Void->Void` and coroutine `Void->Unit` function the coroutine one should be used if the variable is declared in a coroutine.

A new `VAutoClose` tvar flag could be added to indicate if a declaration is an autoclose one.

## Impact on existing code

There should be no breaking runtime changes with this proposal, but the introduction of `autoclose` may cause issues with existing code bases with variables of the same name.

## Drawbacks

This auto closing hides the `close` call which can make debugging trickier if you want to step into the function. A large number of `autoclose` variables in a given scope could also lead to many try catches which can be expensive depending on the target.

## Alternatives

The try catch transformation can be done by a build macro and a `:autoclose` metadata quite easily (https://gist.github.com/Aidan63/3ff8baf8cdad659a3bd1750cd751825e), but having it part of the language means it will see greater use and lead to higher quality (less leaky) libraries due to the ease of resource management.

## Unresolved questions

I don't have any strong feelings about the `autoclose` name, so if anyone else has any suggestions for a better / less likely to cause conflicts name then please suggest.

We may want to allow generators to opt out of the try catch transformation as the target may have a native equivilent. E.g. java 8+ has try resources, C# IDisposable / using, and C++ RAII. These assumably perform better than try catches so it may be useful to allow generators to implement using those.

I went for the structural typing way where anything with a Void->Void close function can be used instead of requiring an interface. Many existing classes in the standard library which would benefit from this are tagged as extern classes so it may not be easy go back and have them implement an interface. We do lose the ability to have stuff like `Array<IClosable>` without resorting to structural typing which is dynamic on many targets. I'm open to reconsidering this if anyone feels strongly.