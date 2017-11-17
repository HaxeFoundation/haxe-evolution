# key => value iteration syntax

* Proposal: [HXP-NNNN](NNNN-key-value-iter.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Support easy iteration over key-value pairs in the `for` loop syntax.

(see also: https://github.com/HaxeFoundation/haxe/issues/2421)

## Motivation

Very often when iterating over various collections we want to have both element key/index and its value
as local variables. Right now, there's no special syntax for that in Haxe, so we have to iterate over
keys/indices and extract the value from the collection at each step. This makes code less concise and
declarative and thus more error-prone:

```haxe
for (key in map.keys()) {
    var value = map.get(key);
    trace(key, value);
}

for (index in 0...array.length) {
    var value = array[index];
    trace(key, value);
}
```

What we could have instead is this:

```haxe
for (key => value in map) {
    trace(key, value);
}

for (index => value in array) {
    trace(key, value);
}
```

## Detailed design

### Syntax

As shown in the "Motivation" section above, the proposed syntax would be `for (key => value in collection)`.
This syntax seems logical and consistent with the current map declaration syntax (`[key => value]`).

It's also not a breaking change, because at the moment the only allowed AST node before the `in` is a simple identifier.

### Semantics

For this to work, we introduce a new standard iterable type:

```haxe
typedef KeyValueIterator<K,V> = Iterator<{key:K, value:V}>;

typedef KeyValueIterable<K,V> = {
    function keyValueIterator():KeyValueIterator<K,V>;
}
```

When typing the `for (key => value in collection) {}` expression, we handle it in a similar way as normal iterators, that is:

 1) if `collection` conforms to `KeyValueIterator<K,V>`, generate:
    ```haxe
    while (collection.hasNext()) {
        var tmp = collection.next();
        var key = tmp.key;
        var value = tmp.value;
    }
    ```

 2) if `collection` conforms to `KeyValueIterable<K,V>`, generate:
    ```haxe
    var iterator = collection.keyValueIterator();
    while (iterator.hasNext()) {
        var tmp = iterator.next();
        var key = tmp.key;
        var value = tmp.value;
    }
    ```

 3) otherwise, emit the `Type has no field keyValueIterator` error


### Performance

For the most common case (`for` loop), if implemented properly, key-value iterators will be fully
inlined and temp variables will be optimized away, so it should be at least as good as hand-written
code similar to the one in the "Motivation" section. For example, see [this try.haxe snippet](http://try-haxe.mrcdk.com/#9c3Aa).

For some specific collection implementations, I imagine the key/value iterator could be faster than
iterating over keys and getting the value each time, if the collection can provide pairs directly.

One performance-related concern is the use of structural typing, which can be an issue on static
targets in their current implementation, however this is a more general problem which also applies to
normal iterators, so it's out of scope for this proposal. Still, we might want to provide a standard
optimized implementation of the readonly `KeyValuePair` type implementation that would be used for key/value iterators
instead of `{key:K, value:V}`.

## Impact on existing code

This shouldn't break much: the `key => value` before `in` within `for` is currently forbidden,
and it [doesn't seem](https://github.com/search?l=&q=keyValueIterator+language%3AHaxe&ref=advsearch&type=Code&utf8=%E2%9C%93) like there's any functions named `keyValueIterator` in public Haxe code.

## Alternatives

No real alternative. Of course, one could macro-process every `for` loop with a global `@:build` macro,
but that would be an overkill for such simple feature.
