# Local static variables

* Proposal: [HXP-0015](0015-local-static-variables.md)
* Author: [YellowAfterlife](https://github.com/yellowafterlife)
* Status: [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10477)

## Introduction

Provide syntax for defining static variables within the scope of a specific field.

## Motivation

Sometimes you want to reuse a static variable within a singular method -
perhaps it is a [data structure](https://github.com/YellowAfterlife/sfgml/blob/b7f32f37126d9ab9197d2248693c5a333019b86b/Array.hx#L242)
that you want to reuse an array/buffer so that you allocate the final returned structure "just right",
or [a native function reference](https://github.com/HaxeFoundation/haxe/blob/c4c2d37f80c136e2259485c1d61b0bd8c38fecfe/std/neko/Lib.hx#L202)
that you resolve once on startup
or even just [depth for a recursive toString() call](https://github.com/HaxeFoundation/haxe/blob/c4c2d37f80c136e2259485c1d61b0bd8c38fecfe/std/cs/_std/Array.hx#L33).
Rest assured, it has to be static. And private. Not to be touched by anything else.

Currently most developers (and the standard library) take the approach of naming it something like `__<method>_<var>`,
but there's a limited amount of convenience in doing so, especially as functions grow longer and going back and forth between variable declarations/use location requires utilizing bookmarks to maintain efficiency.

## Detailed design

Suppose we were to recreate HX-JS $bind function in Haxe code,
```haxe
public static function bind(o:Dynamic, m:Dynamic) {
  if (m == null) return null;
  static var closureID = 0;
  if (m.__id__ == null) m.__id__ = closureID++;
  // ... other code
}
```
this would compile as if it were
```haxe
static var __bind_closureID = 0;
public static function bind(o:Dynamic, m:Dynamic) {
  if (m == null) return null;
  if (m.__id__ == null) m.__id__ = __bind_closureID++;
  // ... other code
}
```

Accessing local variables of containing method in initializer expression should be forbidden,  
```haxe
public static function some(i:Int) {
  static var trouble = i; // <- illegal
  static var trouble2 = function() return ++i; // <- also illegal
}
```
Like with normal statics, either a value or an explicit type would be needed for the static variable.

Simulating block scoping
```haxe
public static function some() {
  // ...
  if (condition) {
    static var one = 1;
  }
  return one; // <- ilegal
}
```
would be consistent with local variables and preferred for situations where a variable is only really needed within a specific branch.

## Impact on existing code

Currently `static` keyword is entirely forbidden inside method bodies so there should be no impact.

## Alternatives

[As an old saying goes](https://yal.cc/wp-content/uploads/2019/03/haxe-macros.jpg),
most syntactic omissions can be corrected with a macro, and this is no exception,
but it would be preferred to not have to iterate the expression tree build-time
(as practice shows, this slowly adds up).

Still, an example of such a macro (implementing `@:static var`) can be found [here](https://github.com/YellowAfterlife/sfhx/blob/master/sf/macro/LocalStatic.hx).

## Unresolved questions

Similarly allowing local `static function` could be handy for platforms where there is a cost to creating/invoking closures.

