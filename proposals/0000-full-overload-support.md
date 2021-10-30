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
If not, then any overloaded function will be renamed.
Normal functions are renamed by their type signature.
For example: 
```hx
overload static function fromArray(arr:Array<Float>):Color
overload static function fromArray(arr:Array<Int>):Color
```
would become
```hx
static function fromArray_array_float_color(arr:Array<Float>):Color
static function fromArray_array_int_color(arr:Array<Int>):Color
```
This applies to function calls as well.

For constructors, one must be selected as the "true" constructor, and others are transformed into static functions.
The true constructor is the one that otherwise would end up with the longest function name. For example: 
```hx
overload function new(r:Int, g:Int, b:Int, a:Int) 
overload function new(r:Int, g:Int, b:Int) 
```
would become
```hx
function new (r:Int, g:Int, b:Int)
static function new_int_int_int(r:Int, g:Int, b:Int):Color
```

For all type signatures, it's calculated by lowercasing the types for each in the args and the return type, then placing them in order with underscores, with return type at the end.
The signature stringification can be changed a bit (especially considering type parameters) but it doesn't matter that much as long as it's unique. Generic Classes also apply like this, but using the given name. 
 
## Impact on existing code

This shouldn't affect any existing code except for making it more cross platform.

## Drawbacks

On the compiled end the function names could be a bit verbose, however that doesn't matter much for most languages.


## Alternatives

You can name the functions this way yourself, but this would make it more convenient to make overloads.


## Unresolved questions

The naming scheme of the type signatures still needs to be fully worked out. It must be unique for any unique signature. 
