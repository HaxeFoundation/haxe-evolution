# Generator Expressions

* Proposal: [HXP-NNNN](NNNN-generator-expressions.md)
* Author: [Nathan Phillip Brink](https://github.com/binki)

## Introduction

Allow the body of the existing [Array Comprehension](https://haxe.org/manual/lf-array-comprehension.html)
syntax to be used as an `Iterator<T>`.

```haxe
class MyClass {
    public function iterate():Iterator<Int> {
        return (for (i in 0...4) if (i % 2 == 0) 2*i);
    }
}
```

## Motivation

Building an `Iterator<T>` currently requires writing implementations
for the `next` and `hasNext` keys. This is easy enough, but the body
of an Array Comprehension is easier to write in some situations and
one might be tempted to just write `[for (x in 0...4) 2*x].iterator()`.
This is fine for simple programs, but sometimes lazy evaluation is
desired.

In Python, the body of an Array Comprehension is nothing special—it is
a first-class “generator expression”. It would seem natural for Haxe
to add generator expressions to complement its existing Array
Comprehension syntax.

## Detailed design

Given how `switch` and `if` are expressions in Haxe and `for`,
`while`, and `do` are currently always `Void`, this could be
implemented by making `for`, `while`, and `do` into expressions
yielding `Iterator<T>`. The yielded `Iterator<T>` should yield the
same sequence of values as the equivalent Array Comprehension syntax
would place in an array if it is iterated immediately. It should
execute lazily, only running as much code as required to satisfy the
current `next()` or `hasNext()` call (`hasNext()` will cause execution
of an iteration of the loop if necessary and precalculate the value
that `next()` would return, `next()` will cause execution of an
iteration of the loop only if there was no intervening `hasNext()`
call).

However, `for`, `while`, and `do` are most commonly used as actual
flow control constructs. The loop should execute immediately and yield
`Void` rather than `Iterable<T>` if:

* `T` is `Void`

* the expression is not used as an rvalue (is this possible to
  detect? Better way?)

* the expression is not immediately preceeded by a “local” opening
  paren `(`. This way, you can safely write loops in `switch` or `if`
  statements as long as you do not wrap the `for`, `while`, or `do` in
  parens. (Is there a better option?)

This change would enable this to work as expected:

```haxe
class Test {
    static function main() {
        // Consume a compiler-generated Iterator<T>
        for (x in getIter()) {
            trace(x);
        }
    }

    // Return an iterator!
    static function getIter() {
        return (for (i in 0...4) 2*i);
    }
}
```

You can pass an iterator (I don’t think that the paren in `exists(`
should count when requiring a paren prior to the expression to evoke
this behavior, but I’m not sure how to say that):

```haxe
Lambda.exists((for (i in 0...4) if (i % 3 == 0) i));
```

This should fail as there is not a “local” paren prior to `for`,
causing the loop to be executed immediately. It will be a compilation
failure as `Lambda.exists()` would not accept `Void`:

```haxe
Lambda.exists(for (i in 0...4) if (i % 3 == 0) i);
```

These loop constructs should execute as loops instead of yielding
`Iterator<T>`s because they are not immediately preceeded by opening
parens:

```haxe
switch (x) {
    case 1:
        for (i in 0...2) {
            doSomething();
        }
    default:
}
```

This `switch` should yield `Iterator<T>` because the bodies of the
`case`s are surrounded by parens:

```haxe
switch (x) {
    case 1: (for (i in 0...2) doSomething());
    default: null;
```

This array should resolve using the existing Array Comprehension
syntax:

```haxe
var a = [for (i in 0...3) i];
```

This array’s elements should be iterators because a paren was
used. The array’s type should be `Array<Iterator<Int>>`:

```haxe
var a = [(for (i in 0...3) i)];
for (x in a[0]) {…}
```

### Code That Would Benefit

This would be helpful whenever trying to write an iterator which
consumes another iterator and does some transformation or filtering.

This would provide a lazy alternative to the `Lambda` utilities which
return `List<T>`.

When implementing `keys()` and `iterator()` members of `IMap<T>`, this
syntax would be useful, with this given monstrous example:

```haxe
class KeyTransformingMap<KSource, KDest, V> implements haxe.Constraints.IMap<KDest, V> {
    var map:haxe.Constraints.IMap<KSource, V>;
    var transformKey:KSource->KDest; // Forward mapping
    var untransformKey:KDest->KSource; // Reverse mapping, yields null if inaccessible value

    public function keys() {
#if generator_expressions
        var untransformedKey;
        return (
            for (key in map.keys()) {
                if ((untransformedKey = untransformKey(key)) != null) {
                    untransformedKey;
                }
            }
        );
#elseif lazy_developer_unlazy_code
        // The developer might consider just returning [«expression»].iterator() instead
        // of trying to implement a lazy iterator. There goes lazy evaluation. Note that
        // for some reason the current Array Comprehension syntax requires you to hoist
        // temporary variables manually.
        var untransformedKey;
        return [
            for (key in map.keys()) {
                if ((untransformedKey = untransformedKey(key)) != null) {
                    untransformedKey;
                }
            }
        ].iterator();
#else
        // If the developer is not lazy, they have to write more code. It is easier
        // to make a mistake and harder to read.
        var wrappedIterator = map.keys();
        var next;
        function advance() {
            next = null;
            while (next == null && wrappedIterator.hasNext()) {
                next = untransformKey(wrappedIterator.next());
            }
        }
        advance();
        return {
            hasNext: function () {
                return next != null;
            },
            next: function () {
                // For some reason, Haxe-3.4rc1 and Haxe-3.3 compiles this to “advance(); return next;”
                // which breaks it… If I manually fix the emitted JavaScript, this works correctly.
                var cur = next;
                advance();
                return cur;
            }
        };
#end
    }
…
}
```

## Impact on existing code

This will break any existing code which places any `while`, `do`, or
`for` in parens if their bodies would not resolve to `Void`. With
current Haxe, doing immediately executes the loop and causes the
`switch` to resolve to `Void`. With the new rules, the `switch` would
resolve to an `Iterator<Int>` without executing the body of the loop,
resulting in a behavior change. *This doesn’t need to be inside of a
`switch` or other construct, it would even affect a parenthesized
statement-level expression, but I expect existing code to do this
inside of `switches` for some reason*:

```haxe
class Test {
    static function main() {
        switch (2) {
            case 2: (
                for (i in 0...3) {
                    f(i);
                }
            );
        }
    }

    static function f(i) {
        trace(i);
        return i;
    }
}
```

## Drawbacks

This will tie down the resolved type of `for`, `do`, and `while` when
used as rvalues, limiting future creativity with those constructs.

## Alternatives

One can always implement iterators manually or use a library or macro
to achieve similar results. E.g., one might use the transducer
pattern. However, the existing built-in Array Comprehension syntax is
already well-defined, understood, and clear. The inability to use this
syntax to build `Iterator<T>` seems like a missing/overlooked feature.

One mentioned alternative is just doing `[for (i in 0...4) i].iterator()`,
but that is not lazily evaluated. Lazy evaluation is sometimes
necessary for scalability.

It might be much more flexible and futureproof to provide a `yield`
feature and allow `yield` to be used within an Array Comprehension
expression. This would allow arbitrarily complex logic to be used
instead of restricting the body of the `Iterator<T>` to have one leaf
expression-statement (like it seems to now). Such a feature could more
easily open the way for the sorts of generator-based hacks found in
JavaScript such as generator-based async/await. But there is no
proposal for this yet.

## Unresolved questions

How to reliably determine if a `for`, `do`, or `while` should resolve
to an `Iterator<T>` versus be immediately executed and resolve to
`Void`? Currently, I suggest extra parens for that, but that might not
be reliable or clearest.
