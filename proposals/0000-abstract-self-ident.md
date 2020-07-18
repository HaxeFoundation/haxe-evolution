# Self access for abstracts

* Proposal: [HXP-NNNN](NNNN-abstract-self-ident.md)
* Author: [Mark Knol](https://github.com/markknol)

## Introduction

Add a way to access "self" for abstracts, which is a getter for `(cast this:MyAbstract)`.

Based on [this discussion](https://github.com/HaxeFoundation/haxe/issues/8162) I'd like to propose an optional `as <ident>` for abstracts, which allows to add/name your own "self":

```haxe
abstract MyAbstract(MyType) as selfIdent {
  
}
```

## Motivation

I find myself copy/pasting the following piece of snippet quite a lot, when dealing with abstracts:
```haxe
var self(get, never):MyAbstract;
inline function get_self() return (cast this:MyAbstract);
```
It would be nice if there was a way to solve this in the language.

#### Use case 1 
One actual example use-case would be this simplified abstract (based on [hx-vector2d](https://github.com/markknol/hx-vector2d/blob/master/src/geom/Vector2d.hx#L89) library).  
Test on https://try.haxe.org/#5979b (which obviously has handwritten `self`)

```haxe
private typedef Vector2dImpl = { x:Float, y:Float }

@:forward
abstract Vector2d(Vector2dImpl) as self from Vector2dImpl {
    public inline function new(x:Float = 0.0, y:Float = 0.0) {
		this = {x: x, y: y};
	}
    
	public inline function inRange(vector:Vector2d, range:Float):Bool {
		// `self` is the current abstract instance, so can use custom operators
		return (self - vector).length < range * range;
	}
    
 	@:op(A - B) public inline function substract(vector:Vector2d):Vector2d {
		return clone().substractAssign(vector);
	}
    
	@:op(A -= B) public inline function substractAssign(by:Vector2d):Vector2d {
		this.x -= by.x;
		this.y -= by.y;
		return this;
	}
    
	public var length(get, never):Float;
	private inline function get_length():Float return this.x * this.x + this.y * this.y;
    
	public inline function clone() return new Vector2d(this.x, this.y);
}
```

#### Use case 2 : enum abstracts
Another useful case would be enum abstracts. Without `as item`, it would switch on `String` which requires you to add a `default` case.

```haxe
enum abstract Item(String) as item {
  var Foo;
  var Bar;
  
  public function getAlternativeName():String {
    return switch item { // pattern matching works on enum values
      case Foo: "fooo!!"
      case Bar: "barr!!"
    }
  }
}
```

## Detailed design

> `abstract MyAbstract(MyType) as <ident> { }`

This feature is only for abstracts. You can name `<ident>` however you want, with exception of existing keywords and it should take field name validation in account. 
Also, `as <ident>` is completely optional for abstracts.

If there is already a field with same name as the ident in the scope there should be a duplication error.

```haxe
abstract MyAbstract(MyType) as foo { // line 1: Duplicate abstract field declaration : MyAbstract.foo
  public inline function foo():Void; 
}
```

At the moment, if you forward a field/function and create a same named field,
there is no duplication/override error. I think we don't have to add error messages for that.

## Impact on existing code

Since it is a new feature and is optional, it doesn't break existing code. 

## Drawbacks

This proposal allows you to name your own ident which is nice and flexible (it allows self being to forwarded from another abstract?) but it doesn't encourage a standard way of writing.
This downside is maybe minor, but is one to consider.

## Alternatives

Add a `self` (or a different name) keyword to abstracts which does the same. 
This would solve the drawbacks, but could break existing code because the keyword could exist in codebases.

## Opening possibilities

-

## Unresolved questions

-
