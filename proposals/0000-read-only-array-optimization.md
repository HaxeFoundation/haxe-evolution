# ReadOnlyArray optimization

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Kim Lindblom](https://github.com/Drakim)

## Introduction

The Haxe compiler is already great at reasoning and optimizing about class properties. It would be great if it could do likewise for ReadOnlyArray, provided that it's declared with the final keyboard and thus unchangable.

## Motivation

Constant LUT (Lookup Tables) would greatly benefit from this. When the compiler is able to deduce index access at compile-time, it can substitute the value in-place just like it would with a regular static field inline optimization. When it can't deduce the value, it falls back to the runtime index access. It would be super useful for all kinds of LUTs, like pre-calculated Math values, and even more amazingly, LUTs of inline functions.

## Detailed design

```haxe
    // Example 1
    static public final arr:ReadOnlyArray<Int> = [10, 20, 30, 40];
    static public function main() {

        var foo = arr[1]; // compiles to var foo = 20;

        var bar = arr[Std.random(2)]; // compiles to var bar = arr[Std.random(2)];

    }

    // Example 2
    static public final arr:ReadOnlyArray<Void->Void> = [bar1, bar2, bar2];
    static public function main() {

        inline arr[1](); // compiles to trace('mega bar');

        inline arr[Std.random(2)](); // compiles to arr[Std.random(2)]();

    }

    static public function bar1() {
        trace('normal bar');
    }

    static public function bar2() {
        trace('mega bar');
    }

    static public function bar2() {
        trace('maximum bar');
    }

```



## Drawbacks

Might we need to worry about the programmer using unsafe casts to modify the array at runtime, or is that justifiably undefined behavior?
