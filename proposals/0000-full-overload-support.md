# Full Overload Support

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [BulbyVR](https://github.com/TheDrawingCoder-Gamer)

## Introduction

Makes the overload keyword fully work between all languages, by creating aliases on the compiled language end.

## Motivation

Right now, the overload keyword isn't fully cross compatible, which defeats the purpose of "Haxe, the cross-platform language." 
This proposal reworks compiling overloading on languages that don't fully support it, like hashlink, by editing the names of
functions and function calls for different function signatures. 
## Detailed design

If the target already supports overloading, that implementation is used. 
If not, then any overloaded function will be renamed, and an adapter function will be created to support dynamic dispatch. 
Normal functions are renamed by their type signature, and prefixed with "hx__"
For example: 
```hx
overload function fromArray(arr:Array<Float>):Color
overload function fromArray(arr:Array<Int>):Color
```
would become
```hx
function fromArray(arr:Dynamic) {
    switch (Type.typeof(arr)) {
        case TClass(Array<Float>): 
            hx__fromArray_Array_Float(arr);
        case TClass(Array<Int>):
            hx__fromArray_Array_Int(arr);
    }
}
function hx__fromArray_Array_Float(arr:Array<Float>):Color
function hx__fromArray_Array_Int(arr:Array<Int>):Color
```
This applies to function calls as well.

For constructors, all are transformed into static functions and the original constructor is transformed into an adapter function. In the overloads, any super calls are redirected to their proper overload. Parent constructors that are not covered by their child are not exposed (can't be called from child). In the overloads, the first argument becomes "inst", meaning instance. It replaces any usage of self and is used for any assignment of member vars.
```hx
class Parent {
    overload function new(r:Int, g:Int, b:Int)
    overload function new(arr:Array<Int>)
}
class Child extends Parent {
  overload function new(r:Int, g:Int, b:Int, a:Int) {
    // unrelated code
    super([r, g, b, a]);
    // blah blah
  }
  overload function new(r:Int, g:Int, b:Int) 
}

```
would become
```hx
class Parent {

  static function hx__new_Int_Int_Int(inst:Parent, r:Int, g:Int, b:Int):Void
  static function hx__new_Array_Float(inst:Parent, arr:Array<Int>):Void
  // Types aren't compatible thus dynamic
  function new(...rest:Dynamic) {
      switch (rest.length) {
          case 1: 
              hx__new_Array_Float(this, rest[0]);
          case 3: 
              hx__new_Int_Int_Int(this, rest[0], rest[1], rest[2]);
      }    
  }
  
}

class Child extends Parent{
    static function hx__new_Int_Int_Int_Int(inst:Child, r:Int, g:Int, b:Int, a:Int):Void {
        // dynamic dispatch isn't required here as we know our parent
        Parent.new_Array_Float(inst, [r, g, b, a]);
    }
    static function hx__new_Int_Int_Int(inst:Child, r:Int, g:Int, b:Int):Void
    // all args are int thus we don't need to make it dynamic
    function new(...rest:Int) {
        switch (rest.length) {
            case 3: 
                hx__new_Int_Int_Int(this, rest[0], rest[1], rest[2]);
            case 4: 
                hx__new_Int_Int_Int_Int(this, rest[0], rest[1], rest[2], rest[3]);
        }
    }
    
}
```

When overriding a function that has overloads, the child function must be overloaded. Any parent functions not covered by the child is exposed.

For all type signatures, it's calculated by stringing the argument types together with underscores.
The signature stringification can be changed a bit (especially considering type parameters) but it doesn't matter that much as long as it's unique. Generic Classes also apply like this, but using the given name. 
If two classes have the same name but are in different packages, they become fully qualified if the function names would overlap.

```hx
overload function fromVector3(v3:bulby.Vector3)
overload function fromVector3(v3:peote.Vector3)
```
becomes
```hx
function hx__fromVector3_bulby_Vector3(v3:bulby.Vector3)
function hx__fromVector3_peote_Vector3(v3:peote.Vector3)
function fromVector3(u0:Dynamic) {
    switch (Type.typeof(u0)) {
        case TClass(bulby.Vector3): 
            hx__fromVector3_bulby_Vector3(u0);
        case TClass(peote.Vector3): 
            hx__fromVector3_peote_Vector3(u0);
    }
}
```


Interfaces must not be allowed to have overloads, and this precludes the class implementing the function from overloading the function. When using reflection, getProperty must be used. Return types being changed does not count as a different type signature, and will emit an error. 

How do function calls know what to call? 

Haxe already knows how to look through overloads when it comes to monomorphs; But for clarity, Haxe will look through the overloads in order.
It first removes any that can't be the same length as the amount of args given, taking optional arguments into consideration. 
Then, it tries to find one where the first arg unifies with the overload's first type, and then second arg, and so on. 
Monomorphs are calculated exactly like they are for overloads already. 

## Impact on existing code

This shouldn't affect any existing code except for making it more cross platform.

## Drawbacks

On the compiled end the function names could be a bit verbose, however that doesn't matter much for most languages.


## Alternatives

You can name the functions this way yourself, but this would make it more convenient to make overloads.


## Unresolved questions

The naming scheme of the type signatures still needs to be fully worked out. It must be unique for any unique signature. 
