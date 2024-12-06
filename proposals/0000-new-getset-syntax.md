# New getter & setter syntax

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [GasInfinity](https://github.com/GasInfinity)

## Introduction

Provide new syntax to make properties

```haxe
private var _testField:T;
public var testProperty:T { get -> _testField; set -> _testField = value; }
```

## Motivation

Every time you want to make properties in Haxe, you need to create the functions for these getters and setters. It's quite verbose. I propose a new way to make properties inspired in the C# syntax:

```haxe
// Creates a backing field and works like a field, can be overriden if a class inherits this property
public var propertyWithField:T { get; set; }

// Creates a backing field, but this property can only be assigned once in the constructor
public var readonlyProperty:T { get; }

// Raw property, doesn't create a backing field
public var property:T { get -> {} set -> {} }
```
  vs
```haxe
// Creates a field and works like a field, can be overriden if a class inherits this property
private var _propertyWithField;
public var propertyWithField(get, set):T;

function get_propertyWithField():T return _propertyWithField;
function set_propertyWithField(value:T):T return _propertyWithField = value;

// Creates a field, but this property can only be assigned once in the constructor
private final var _readonlyProperty;
public var readonlyProperty(get, never):T;

function get_readonlyProperty():T return _readonlyProperty;

// Raw property, doesn't create a backing field
public var property(get, set):T;

function get_property():T {}
function set_property(value:T):T {}
```

It could achieve the same results with less code while maintaining readability.

## Detailed design

### Syntax

The following syntax is proposed for properties:

* Raw Property
```haxe
public var name:Type { get -> expr set -> expr }

// The equivalent of this code is:

public var name(get, set):Type;

function get_name():Type expr
function set_name(value:Type):Type expr
```

* Property with only a getter:
```haxe
public var name:Type { get; }

// This doesn't have a 1 to 1 equivalent, but this is somewhat equivalent:

private var _name:Type;
public var name(get, never):Type;

function get_name() return _name;
```

* Autoproperty:
```haxe
public var name:Type { get; set; }

// The equivalent of this code is:

private var _name:Type;
public var name(get, set):Type;

function get_name():Type return _name;
function set_name(value:Type) return _name = value;
```

### Behaviour

* A property will always create internally functions for getters and setters because every setter and getter can be overridden.
    ```haxe
    class A
    {
        public var property:Int { get; } // An instance of the class A can only get the value
    }

    class B extends A
    {
        public var property:Int { get; set; } // An instance of the class B can get and set the value
    }

    class C extends A
    {
        public var property:Int { override get -> 1; }
    }

    function main()
    {
        var b:B = new B();
        var a:A = b;
        // a.property = 20; // Error, property doesn't have a setter
        b.property = 20; // No errors

        var c = new C();
        // c.property = 20; // Error, property doesn't have a setter

        trace(c.property); // Output: 1

    }
    ```
* A property can have access modifiers
    ```haxe
    class A
    {
        public var property:Int { inline get; private set; }
        // private var otherProperty:Int { public get; set; } // Error, the visibility of a getter or setter must be lower than the visibility of the property itself

        public function modifyProperty():Void
        {
            property = 20; // No errors
        }
    }

    function main()
    {
        var a = new A();
        // a.property = 20; // Error, can't set property, it's private
        a.modifyProperty();

        trace(a.property); // This gets inlined
    }
    ```
* A property can be initialized
    ```haxe
    class A
    {
        public var property:Int { get; } = 40;
    }

    function main()
    {
        var a = new A();

        trace(a.property); // Output: 40
    }
    ```
* A property with only a getter can be assigned only once in the constructor
    ```haxe
    class A
    {
        public var property:Int { get; }

        public function new(propertyValue:Int)
        {
            this.property = propertyValue;
        }
    }

    function main()
    {
        var a = new A(600);

        trace(a.property); // Output: 600
    }
    ```
* The first argument passed to the setter function has the implicit argument name of `value`
    ```haxe
    class A
    {
        private var field:Int;
        public var property:Int {
            get -> {
                return field;
            }
            set -> {
                return field = value;
            }
        } // This could also be written in a short form { get -> field; set -> field = value; }

    }

    function main()
    {
        var a = new A();
        a.property = 10;

        trace(a.property); // Output: 10
    }
    ```
* A property can be static
    ```haxe
    class A
    {
        public static var property:Int { get; set; }
    }

    function main()
    {
        A.property = 20;

        trace(A.property); // Output: 20
    }
    ```
* A getter or setter without a function implementation and with the `final` modifier in a property should behave like a normal field access.
    ```haxe
    class A
    {
        public var property:Int { final get; set; }
    }

    class B extends A
    {
        @:isVar
        public var property:Int { override set -> property = value; } // Pointless, but we're just showing what can be done or not
    }
    
    function main()
    {
        var a = new A();
        var b:B = a;

        a.property = 20;

        trace(a.property); // Normal field access because of the final modifier without function implementation
        b.property = 30; // This would call the function of the B class

        trace(b.property); // Still normal field access
    }
    ```
* A property can be declared in an `interface`, but it cannot have any implementation, classes that extend that interface must implement the property (As an Autoproperty or as a raw property)
    ```haxe
    interface A
    {
        var property:Int { get; }
    }

    interface B extends A
    {
        var property:Int { get; set; } // This works, we're only declaring that the property has a getter and a setter
        // var otherProperty:Int { get -> 1; } // Error, a property in an interface cannot have any implementation
    }

    class C implements B // Error, missing implementation of the getter & setter of the property 'property'
    {

    }
    ```
* A property in an `abstract class` can have abstract getters/setters
    ```haxe
    abstract class A
    {
        var property:Int { abstract get; }
    }

    class B extends A // Error, B must implement the getter of the property 'property'
    {
    }
    ```
* A property in an `abstract` must have implementations
    ```haxe
    abstract A(Int)
    {
        // public var badProperty:Int { get; set; } // Error, the getters/setters in an abstract must have a function implementation

        public var goodProperty:Int { inline get -> 1; } // Good, we have a function implementation
        public var otherGoodProperty:Int { inline get -> this; inline set -> this = value; } // Good, we have only function implementations
    }
    ```
* A property can be overriden in a derived class, but it must be overriden as a raw property. 
    ```haxe
    class A
    {
        public var property:Int { get; set; }
    }

    class B extends A 
    {
        // Allowed, we're making function implementations for the property
        public var property: Int { override get -> 1; override set -> -1; }

        // Maybe we could use the @:isVar metadata to be able to access it's backing field from the class
        // @:isVar
        // public var property: Int { override get -> property * 2; override set -> property = value / 2; }
    }

    /* Not allowed, it must have an implementation
    class C extends A
    {
        public var property:Int { override final get; override final set; }
    }*/

    function main()
    {
        var a = new A();
        var b = new B();

        trace(a.property); // Output: 0 (default value of Int)
        trace(b.property); // Output: 1

        a = b;
        trace(a.property = 2); // Output: -1
        trace(a.property); // Output: 1
    }
    ```
* If the property has the `@:isVar` metadata, it can assign to the backing field generated by the compiler when assigning to itself:
    ```haxe
    @:isVar
    public static var property:Int { get -> property; set -> property = value; }
    ```

### Final notes

If this gets implemented in any way, we could soft deprecate the old property syntax and move to the new one.
So, in the next major version, one breaking change might be properties.

## Impact on existing code

This is new syntax so I don't think it will break existing code. 
*Unless in the next major version we remove the old property syntax. That would be a very **big** breaking change.*

## Drawbacks

No drawbacks. *I think, correct me if I'm wrong, please*

## Alternatives

We could do it in many other ways, like:
```haxe
public var property:T { get => expr; set => expr; }
```

But I think that we should maintain the same syntax used for arrow functions, and, instead of doing:
```haxe
public var property:T { () -> expr; (value) -> expr }
```
or
```haxe
public var property:T { get() -> expr; set(value) -> expr }
```

We remove the parenthesis and get a syntax similar to the C# property syntax but using the Haxe `->` used for functions

## Unresolved questions

* Should we allow only setters without getters like now? (The current property syntax allows making a property with only a setter)
    ```haxe
    class A
    {
        public var property:Int { set; } // Error here?
    }

    function main()
    {
        var a = new A();
        a.property = 20;

        // trace(a.property); // or Error here?
    }
    ```
    ```haxe
    // The current property syntax allows making only a setter without a getter

    class A
    {
        @:isVar
        public var currentProperty(never, set):Int;

        function set_currentProperty(value:Int):Int return currentProperty = value;
    }

    function main()
    {
        var a = new A();
        a.currentProperty = 20;

        // trace(a.currentProperty); // Error
    }
    ```

