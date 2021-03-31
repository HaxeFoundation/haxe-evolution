# `enum abstract` over `enum`

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Kevin Leung](https://github.com/kevinresol)

## Introduction

`enum abstract` is basically a mechansim to define a subset from a superset.
For example, when used over `Int`, it creates a finite subset from a infinite superset which consists of all integers. (Well, techinically it is not really infinite because it is limited by computer memory)

Therefore in theory, it could also be used over ordinary `enum`, to create a subset of its constructors.

## Motivation

1. Expose `enum` partially

    ```haxe
    enum CRUD {
    	Create(id:String, data:Any);
    	Read(id:String);
    	Update(id:String, data:Any);
    	Delete(id:String);
    }
    
    enum abstract ReadAndUpdate(CRUD) to CRUD {
    	final ReadData = Read;
    	final UpdateData = Update;
    }
    
    function doAdminTask(task:CRUD):Void {
    	// perform the task
    }
    
    function doEditorTask(task:ReadAndUpdate):Void {
    	return doAdminTask(task);
    }
    ```

    In the above example, we can expose a limited set of operations to restricted users via the `doEditorTask`

2. Reuse existing `enum` to reduce code size

    ```haxe
    enum abstract Status<T>(Option<T>) to Option<T> {
    	final Continue = Some;
    	final End = None;
    }
    ```

    Since at runtime `Status` does not exist and it will be represented by `Option`. We saved the code size for it.

3. Enable enum instance/static fields

    We get https://github.com/HaxeFoundation/haxe-evolution/issues/10 for free


## Detailed design

To start simple, we should only allow declaring an alias to the underlying enum constructor. Since the type of the abstract fields is just the same as the underlying aliased enum constructor, pattern matching should just work with minimal work.

#### Futher developments:

- Partial/full application of enum constructors:

    ```haxe
    enum abstract MaybeValue<T>(Option<T>) to Option<T> {
    	final One = Some(1);
    	final Two = Some(2);
    	final Other = Some;
    	final Nothing = None;
    }
    ```

    This may be handled in conjunction with other evolution proposals such as https://github.com/HaxeFoundation/haxe-evolution/pull/86

## Impact on existing code

Since the abstract fields are no longer a primitive value, it may affect macro code that tries to obtain the literal value at compile time.
But since the feature is new, such breakage will only happen when a abstract-enum-over-enum is passed to such macros.

## Drawbacks

To be discussed.

## Alternatives

One can always declare a separate ordinary enum as a subset of another enum, but that would require manual translation to its superset. Also it incurs extra generated code size.

## Unresolved questions

To be discussed.
