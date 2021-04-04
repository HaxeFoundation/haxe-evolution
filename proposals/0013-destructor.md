# Destructor

* Proposal: [HXP-0013](0013-destructor.md)
* Author: [Khalyomede](https://github.com/khalyomede)

## Introduction

Many languages that Haxe compile to support the concept of class destructor:

- PHP
- C#
- Java
- C++
- ...

This proposal is about adding support for class destructor.

## Motivation

### The goal of a destructor

Destructors are primarily used to free open or mobilized resources during a object instanciation or runtime. Trying to achieve this currently is not possible because we do not have access to a hook for knowing when an object is going "out of scope" (understand: the object within its scope is terminated and will be garbage collected). Other languages do not provide this kind of hook but expose a destructor developers can override to perform their own logic.

### Example of usage

An example of usage for destructors can be seen using the filesystem. Let us say you are modelizing a log file:

```haxe
import sys.io.FileSystem;

final class LogFile {

  var fileInput: Null<FileInput>;

  public function new() {
    this.fileInput = FileSystem.read("storage/logs/app.log");
  }

  public function getContent(): String {
    if (this.fileInput != null) {
      return this.fileInput.readAll().toString();
    }

    return "";
  }

  public function close() {
    if (this.fileInput != null) {
      this.fileInput.close();
    }
  }
}
```

Developers can then get the content of their log file like so:

```haxe
final class Main {

  public static function main() {
    final logFile = new LogFile();

    trace(logFile.getContent());
  }
}
```

I did not closed the file on purpose. In this case, no matter wether the file was closed before the end of the program, we would like to add this extra layer of caution in order to avoid letting a read cursor opened. The constructor is perfectly suited to perform this kind of "house keeping" task.

Some others things we could do in a destructor:

- closing a database connection
- storing analytics data

### Is Destructor needed by other Haxe developers?

Some Haxe developers have searched or requested this feature:

- [StackOverflow: Haxe: define a function/macro which fires when an object goes out of scope?](https://stackoverflow.com/questions/31572190/haxe-define-a-function-macro-which-fires-when-an-object-goes-out-of-scope)
- [hxcpp Github: Feature request. Disabling GC](https://github.com/HaxeFoundation/hxcpp/issues/710)
- [Google Groups: GC, refcounting, weak references, destructors and observers](https://groups.google.com/g/haxelang/c/ZJLuUHKQ-qU?pli=1)

## Detailed design

### Example of implementation

Taking our log file example, this is how we would implement it using the constructor method:

```haxe
final class LogFile {
  
  // ...

  public function destructor() {
    this.close();
  }
}
```

### Destructor method signature

The signature of this method is the following:

```haxe
public function destructor(): Void;
```

It takes no arguments, and must return nothing. It should only perform tear down tasks, and can use all the methods and properties of the instanciated object.

### Behavior rules

- There can be only one destructor per class. 
- The destructor method cannot be static. 
- The destructor must be called whenever the instanciated object is freed from its current scope.

Consider this example:

```haxe
final class Main {

  public static function main() {
    Main.displayContent();
  }

  public static function displayContent() {
    final logFile = new LogFile();

    trace(logFile.getContent());

    // At this moment, logFile goes out of scope, it will be freed and its destructor must be called.
  }
}
```

Versus this example:

```haxe
final class Main {

  public static function main() {
    final logFile = new LogFile();

    Main.displayContent(logFile);

    // The scope of the logFile variable is finishes here, call its destructor now.
  }

  public static function displayContent(logFile: LogFile) {
    trace(logFile.getContent());

    // Not calling its destructor yet, since its scope is not finished here.
  }
}
```

- A child class can call its parent destructor using `super()` only if its parent have declared a `super()` method. If not, the compiler must raise a compilation error (in the same fashion we cannot construct an object from a class that do not implement a constructor).
- If the target do not support destructors, the compiler must raise an error.

### Destructor support in Haxe targets

As per my basic knowledge, here is a list of targets with their support for destructors:

| Target      | Support destructor? | Destructor method  |
|-------------|---------------------|--------------------|
| PHP         | ✅                   | `__destruct() {}`  |
| Swift       | ✅                   | `deinit {}`        |
| Objective-C |                     |                    |
| Java        | ✅                   | `finalize() {}`    |
| Neko        |                     |                    |
| NodeJS      |                     |                    |
| Flash       |                     |                    |
| C++         | ✅                   | `~class_name() {}` |
| Python      | ✅                   | `\_\_del\_\_():`   |
| C#          | ✅                   | `~class_name() {}` |
| HashLink    |                     |                    |
| Lua         |                     |                    |

## Impact on existing code

### Breaking change

This proposal is a breaking change, and should be introduced in the next major release because we can't be sure the "destructor" method name is not already used by developers.

### Migration

The migration will not be tedious, but not hard, since it is just a matter of searching for all classes containing a method named "destructor", and refactoring it to rename it for something else. This could be even automatizable, but developers might want to do this refactor manually to provide a new meaningful method name.

## Drawbacks

## Alternatives

### Parent-child

A first alternative would be to extends from a parent class that defines a destructor, so that child classes can re-implement it for their own purpose. This alternative have a drawback: not forgetting to call the parent class "destructor" in the code. As it relies on human capability to be consistant over refactor to not forget it, this is not an ideal solution.

### Manual call

Another alternative would be to just use the destructor method by hand. As the first alternative, it relies on not forgetting to call this method at the correct place, and can introduce some boilerplate code. More code means more chance to introduce bugs of logic.

### Plateform-specific hooks

The last alternative is to rely on plateform-specific termination hooks, like PHP's [`register_shutdown_function()`](https://www.php.net/manual/en/function.register-shutdown-function.php). The big show stopper here is that we loose the versatility of Haxe language over compiler-time flags. This introduce boilerplate code, and is not testable.

```haxe
final class Main {

  public static function main() {
    final logFile = new LogFile();

    #if php
    php.Syntax.code("register_shutdown_function({0})", () -> logFile.close());
    #end
  }
}
```

## Opening possibilities

Adding a conditonal compilation flag for implementing destructors on targets that implements them, in order for targets that do not supports destructors to not generate any code (and no compilation error). Example:

```haxe
final class LogFile {

  #if destructor
  public function destructor() {
    // ...
  }
  #end
}
```

## Unresolved questions

The name of the destructor method is subject to debate. Here is some track for discussing about it:

- One would think of using a short name, to match the `new()` method. Something like `off()` for example?
- If we choose `destructor` to have the most declarative method name possible, should we also introduce an alias to `new()` named `constructor` and deprecate `new()`? 
- If we prefer verbs for methods, we could go for `destroy` or `destruct` (along with `construct`).
