# Onsite getters and setters implementation

* Proposal: [HXP-NNNN](NNNN-onsite-getset-declaration.md)
* Author: [Dmitry Hryppa](https://github.com/haxedev)

## Introduction

Add a way to implement getters and setters directly in the place where they are declared. At least for simple (one line) expressions.

Example:
```haxe
public var isDataExists(() -> data != null, never):Bool;
```

## Motivation

Sometimes we need to implement getters or setters with a very simple implementation, just to encapsulate and protect some data.

Nowadays we have short arrow functions which have been added in Haxe 4. So, it would be nice to have the ability to use them with getters and setters to reduce boilerplate code.

## Detailed design

Proposed "short syntax" of getters and setters implementation may work pretty nice with brand new arrow functions. Let's compare the next examples:
```haxe
public var isDataExists(get, never):Bool;

//and somewhere in the class:
function get_isDataExists():Bool {
	return data != null;
}
```
VS
```haxe
public var isDataExists(() -> data != null, never):Bool;
```

This feature may be done without `() -> expr`, with just an expression inside. But it may be a bit less consistent compared to a full syntax. Example:
```haxe
public var isDataExists(data != null, never):Bool;
```

## Impact on existing code

There may be a problem with overriding getters and setters. But to solve this Haxe compiler may generate from short expression: `() -> data != null` just a regular syntax:

```haxe
function get_isDataExists():Bool {
    return data != null;
}
```

So, `get_isDataExist` will be allowed for overriding by the child.

## Drawbacks

.

## Alternatives

Alternatively, it may be implemented with a macro.
But it is not possible to change the standard syntax and pass expression into `(get, set)` instead of identifiers.

Custom implementation may looks like:
```haxe
//@:set is not provided so it is equal `never`
@:get(data != null)
public var isDataExists:Bool;
```

Would be nice to have this feature out of the box to avoid mixing different syntax of getters and setters in your codebase.

## Opening possibilities

.

## Unresolved questions

.
