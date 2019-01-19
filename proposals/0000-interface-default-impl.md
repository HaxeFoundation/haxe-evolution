# Default functions implementation and default values initialization in the interfaces

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Dmitry Hryppa](https://github.com/dmitryhryppa)

## Introduction
Allow default function implementations in interfaces. As an alternative to create intermediate class.

## Motivation

In a situation when we have an interface which is implemented by a bunch of classes and we want to add a new method into that interface, then we need to change all our classes and implement that method everywhere. It is annoying if we need a method which should have the same implementation for all our classes.
Example:

```haxe
interface IUser {
    public function sayHello ():Void;
    
    //We can't add this new things without impacting all our codebase bellow.
    private var email:String; 
    public function isEmailValid ():Bool;
}

//Now, we need to implement isEmailValid() in all classes
//Also we need to declare email field in all those classes 
class A implements IUser { public function sayHello ():Void {} }
class B implements IUser { public function sayHello ():Void {} }
class C implements IUser { public function sayHello ():Void {} }
```

Currently, the solution  is: 
```haxe
interface IUser {
    public function sayHello ():Void;
}

//We adding new class here for the methods which will have the same behavior for all child classes:
class User implements IUser {
    public var email:String;
    public function new (email:String) {
        this.name = name;
    }
    public function isEmailValid ():Bool {
        /* ... */
        return false;
    }
    
    //Empty. 
    //But it is required by interface.
    //And child classes can forget to implement their own realization
    public function sayHello ():Void {
      
    }
}

//Now, we can call isEmailValid() from superclass.
//Nothing needs to be declared and implemented after it was added into User class.
class A extends User { 
  //But we need to use override keyword here. 
  override public function syHello ():Void {
    //And remove super.sayHello() from here.
  } 
}

//And now, the developer will not be noticed if he will forget to implement required `sayHello()` method.
class A extends User {} 
class A extends User {}
```
To avoid this kind of situation when the developer will forget about 'sayHello ()' implementation, we need to create a new interface which will require 'sayHello()' and will have nothing in common with `User` class.

But we can make our life much easier if it will be possible to add default implementations directly in the interfaces. Similar like it has been done in Java 8.

## Detailed design
Let's look at the example of possible implementation of this feature:

```haxe
interface IUser {
    //Fields marked as default should be initialized here;
    default private var email:String = "";
    
    //Methods marked as default should be implemented here;
    default public function isEmailValid ():Bool {
        /* ... */ 
        //This method has access the interface variables and functions.
        return false;
    }
    
    //This method is still required to be implemented by childs
    public function sayHello ():Void;
}

//All classes already contains `isEmailValid()` method, which implementation was injected from IUser interface.
//All classes already contain `email` field, which is initialized in the interface.
class Student implements IUser {
    public function sayHello ():Void {
        trace ("Hi, this is a hello from A. I'm a student.");
    }
} 
class Teacher implements IUser {
    public function sayHello ():Void {
        trace ("Hi, this is a hello from B. I'm a teacher.");
    }
}
class Cat implements IUser {
    public function sayHello ():Void {
        trace ("Hi, this is a hello from C. I'm a cat.");
    }
}
```

Generated code should be the same as it already is in the current implementation. 
And example code described above can be unrolled into this:

```haxe
interface IUser {
    private var email:String;
    public function isEmailValid ():Bool;
    public function sayHello ():Void;
}

class A implements IUser {
    private var email:String = ""; //<< injected interface 
    public function isEmailValid ():Bool { /*impl from interface */}
    public function sayHello ():Void { /* own impl */}
} 

class B implements IUser {
    private var email:String = ""; //<< injected interface 
    public function isEmailValid ():Bool { /*impl from interface */}
    public function sayHello ():Void { /* own impl */}
}

class C implements IUser {
    private var email:String = ""; //<< injected interface 
    public function isEmailValid ():Bool { /*impl from interface */}
    public function sayHello ():Void { /* own impl */}
}
```


## Impact on existing code

Should not impact existing code.

## Drawbacks

I don't see any drawbacks: on the contrary, it will help to avoid unnecessary, boilerplate code.

## Alternatives

As alternative soultion we need to create new entities in our codebase, thereby increasing the cost of writing the code, reducing its readability.

## Unresolved questions
None.
