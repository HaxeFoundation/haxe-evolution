# Shading

* Proposal: [HXP-0000](0000-shading.md)
* Author: [EliteMasterEric](https://github.com/EliteMasterEric)

## Introduction

Provide the ability to rename and transform package and class names of types during compilation.

## Motivation

In some use cases, there exists a need to produce a set of compiled classes with a specific set of class names.

For example, when compiling with the Java or JVM target to produce a Java module, the resulting code contains the `haxe` package with several classes in it. However, in Java 9, Java does not allow more than one module to contain the same package. This is important because it means you cannot use two Java modules built with Haxe in the same project. This is because they both contain and depend on the `haxe` package, and between Haxe versions, one modules's `haxe` classes may have incompatible differences over the other.

A solution is available for the Java target; when using a build tool such as Gradle or Maven, you may use **shading** to rename the dependency package to one which will be compatible universally. For example, Gradle [shadow](https://github.com/johnrengelman/shadow) and Maven [shade](https://maven.apache.org/plugins/maven-shade-plugin/) are the two widely used tools which allow for this.

I currently use this with the Haxe Java target to rename references to `haxe` to `net.myproject.shadow.haxe`, leaving these class and package names globally unique and allowing me to bundle these dependency classes into the JAR without conflicts.

Here is an example:

```gradle
shadowJar {
  archiveClassifier = ''
  configurations = [project.configurations.shade]

  // Shadow the listed packages.
  relocate 'haxe', "${project.group}.shadow.haxe"
}
```

After the Gradle build completes, opening the JAR reveals that the classes in the `haxe` have all been refactored. All references to the packages or the classes in it have been replaces with references to those same classes, in a different package.

However, as stated in the Haxe Spring Report, the Java source target is currently difficult to maintain for the Haxe team, and the team is looking into deprecating it completely in favor of JVM. However, if this happens, my only options for generating valid modules are:

- Splitting the classes up into a `haxe` standalone module and a specific module for my project code. This results in an extra dependency which would have to be maintained.
- Somehow decompiling the generated JVM code into Java source files, then recompiling them. This is highly error-prone, since Java decompilers often do not create fully compatible code.
- Something insane like manipulating the bytecode directly.

## Detailed design

Users looking to shade classes would be able to do so in two ways:

- A compiler argument: `--shade my.source.pkg=my.target.pkg`
- An annotation: `@:shade("my.new.target.name")`

### Compiler argument syntax

When shading via compiler argument, a single class name will be transformed into another class name, or a package (and all subpackages) will be transformed into another package name.

For example, take this original source:

```haxe
// original Haxe code
import my.source.pkg.MyClass;

var myVariable:MyClass = new MyClass();
```

Compiling to the Java target with no changes, you would get something like:

```java
import my.source.pkg.MyClass;

MyClass myVariable = new MyClass();
```

With `--shade my.source.pkg=my.target.pkg` defined as a command line argument or in a `.hxml` file, the resulting code would be:

```java
import my.target.pkg.MyClass;

MyClass myVariable = new MyClass();
```

With `--shade my.source.pkg.MyClass=my.target.pkg.TheClass` defined as a command line argument or in a `.hxml` file, the resulting code would be:

```java
import my.target.pkg.TheClass;

TheClass myVariable = new TheClass();
```

It is important that `--shade haxe=my.target.pkg.haxe` work as expected, where the generated code which refers to the `haxe` package refers to the new package instead.

### Annotation syntax

Shading could also be allowed via annotations on a class, which could be applied via macros or conditional compilation.

For example, take this original source:

```haxe
// original Haxe code
package my.source.pkg;

class MyClass {}
```

With this modification:

```haxe
// original Haxe code
package my.source.pkg;

@:shade("my.target.pkg.TheClass")
class MyClass {}
```

The file would be generated as `my/target/pkg/TheClass.java`, without affecting the ability of other Haxe code to access it properly.

This could then be applied conditionally:

```haxe
#if release
@:shade("my.target.pkg.TheClass")
#end
```

Or applied via a type building macro:

```haxe
var shadingTarget = '$targetPackage';
Context.getLocalClass().get().meta.add(':shade', shadingTarget);
```

## Impact on existing code

This should not have an impact on existing code, as it will only be applied when the newly created annotation or compiler flag is set.

## Drawbacks

This may have unforeseen impacts on code which relies on the relative location of other classes. For example, shading could move one class such that, in Haxe, they are in the same package, but in the target language, they are not, or vice versa. There is also the matter of determining what should happen if two classes would overlap through this (probably just throw a human readable error at compile time).

## Alternatives

As stated, shading via Gradle or Maven works now, but this would require further maintenance and feature improvements to the Java target, which is currently under consideration for deprecation.

## Unresolved questions

The correct terminology needs to be chosen. Transform, shadow, shade, mutate, mangle could all be valid names.

A more in-depth analysis of the impact of reworking code generation would need to be done.
