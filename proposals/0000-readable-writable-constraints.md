# Readable/Writbale constraints

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [lublak](https://github.com/lublak)

## Introduction

Combines all readable/writable access types for a constraint check. 

## Motivation

This makes easier to work with propertys in constraints.
For a generic class it should not matter what reading looks like.
The main thing is readable and the same applies to writable.

## Detailed design

Example:

```haxe
class TestReadable<T:{@:readable var field:String}> {
  public function new(t:T) {}
}

class TestWritable<T:{@:writable var field:String}> {
  public function new(t:T) {}
}

class ReadableWithNormalField {
  public var field:String;
  public function new() {}
}

class ReadableWithDefaultNull {
  public var field(default, null):String;
  public function new() {}
}

class ReadableWithGetter {
  public var field(get, never):String;
  public function get_field() return 'test';

  public function new() {}
}

class NoReableField {
  private var field:String;
  public function new() {}
}


class WritableWithNormalField {
  public var field:String;
  public function new() {}
}

class WritableWithSetter {
  public var field(default, set):String;
  private function set_field(s:String) {
    // .....
    return s;
  }
  public function new() {}
}

class NoWritableField {
  public var field(default, never):String;
  public function new() {}
}

class Main {
  public static function main() {
    new TestReadable(new ReadableWithNormalField()); // ok
    new TestReadable(new ReadableWithDefaultNull()); // ok
    new TestReadable(new ReadableWithGetter()); // ok
    new TestReadable(new NoReableField()); // Constraint check failure

    new TestWritable(new WritableWithNormalField ()); // ok
    new TestWritable(new WritableWithSetter()); // ok
    new TestWritable(new NoWritableField()); // Constraint check failure
  }
}
```

## Impact on existing code

None.

## Drawbacks

None.

## Alternatives

Currently it is only possible over `@:genericBuild` with limitations.

- nested constraints are very difficult to check
- you need the type directly in the construction (Unknown<0> impossible)
- everytime you wan't to work with it you need a `@:genericBuild` over your class
- `@:genericBuild` cannot be used in macros

## Unresolved questions

Should it be metadatas?
Haxe added the key words `final` or `enum abstract` as an alternative to metadata `@:final` and `@:enum`.