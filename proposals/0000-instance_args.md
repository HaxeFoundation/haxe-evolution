# Constructor instance arguments

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [RblSb](https://github.com/RblSb)

## Introduction

Provide `this.arg` syntax for assigning constructor arguments to instance fields.

## Motivation

This operation is very common in object-oriented programming and with this syntax data-classes should be easier to write, read and change.

## Detailed design

Lets see a typical class example, that is used to keep some data:
```haxe
class People
  final name:String;
  final age:Int;
  function new(name:String, age:Int) {
    this.name = name;
    this.age = age;
  }
}
```
This is how it would look with the new syntax:
```haxe
class People
  final name:String;
  final age:Int;
  function new(this.name, this.age) {}
}
```
With the second example there is less code duplication and no requirement to specify the types, they are derived from respective fields. `this.arguments` are set before all code in the constructor body.
Access to `name` and `this.name` will be identical and mean access to the field `name`, like there isn't any argument variable in the constructor:
```haxe
class People extends Bug {
  final name:String;
  final age:Int;
  function new(this.name, this.age) {
    // after fields are assigned, constructor code starts execution

    age++; // changes this.age

    super();

    name = 'Cooler $name'; // changes this.name
  }
}
```
This should mean less variable shadowing mistakes.

`this.` arguments can be used with normal arguments in any order:
```haxe
function new(type:Int, this.context, isBlock = false) {}
```

If `foo` field for `this.foo` argument doesn't exist in a class, there should be a compiler error.

This syntax is disabled outside of constructor functions.

## Impact on existing code

Should be no impact.

## Alternatives

There is a possible macro code generation replacement, but I think this syntax is simple enough to understand, to be in language itself.

## Opening possibilities

We can omit constructor body requirement, if there is already any `this.argument` in `new` args:

`function new(this.name, this.age);`


## Unresolved questions