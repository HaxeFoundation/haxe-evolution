# Default type parameters

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Ben Merckx](https://github.com/benmerckx)

## Introduction

Optionally declare a default type for generic type parameters.

````Haxe
class Test<T = String> {}
$type((null: Test)); // Test<String>
````

## Motivation

### Use case: types

Some types can benefit from having default type parameters. As an example look at
a generic virtual dom component: 
````haxe
class Component<Props: {}, State: {}> {}
````
Users of the library subclass the Component. Often there's no state or props used,
but with current haxe it's required to write out the empty parameters explicitly:
````haxe
class MyComponent extends Component<{}, {}> {}
````
With defaults it's possible to simply extend and leave the defaults to the library:
````haxe
class Component<Props: {} = {}, State: {} = {}> {}
class MyComponent extends Component {}
class MyComponentWithProps extends Component<MyProps> {}
class MyComponentWithPropsAndState extends Component<MyProps, MyState> {}
````

Another use case is simplifying user facing APIs where some types are only necessary 
to be given explicitly in very specific cases. The types are ready to be used without
the user having all of the implementation details.

For example tink_core defines an Error as
````haxe
typedef Error = TypedError<Dynamic>;
````
where with default type parameters there would not necessarily have been a distinction
as `Error` and `TypedError` could have been defined as:
````haxe
class Error<T = Dynamic> {}
````

Another example would be a generic Promise implementation which holds error data as well. 
In most cases it makes a lot of sense to default the error type parameter to an error type
that works for the user but would not restrict them from using it differently if the use
case came up.
````haxe
class Promise<T, E = Error> {}
````

Using the dom in the javascript target can also demonstrate the use as accessing elements
is usually done through `js.html.Element`. That works as long as you want access to those
properties. But in a few cases you need access to specific properties of the element and
thus want it typed. Say in a lifecycle method of typical virtual dom components
(ignoring state or props here):

````haxe
class Component<E = js.html.Element> {
  onmount(element: E) {}
}
````
If you'd like to set the `src` property of an image this can be used as:
````haxe
class Image extends Component<js.html.ImageElement>
````
But in most other cases you can use `Component` directly without passing a specific element type.

See also: https://github.com/HaxeFlixel/flixel/issues/1677

### Use case: methods

Methods with a generic type parameter are not always able to infer that type from the parameters (especially if that type is optional).

````haxe
function createMyClass<T = MyDefault>(?input: T): MyClass<T>
  return new MyClass(if (input == null) new MyDefault() else input);

$type(createMyClass()); // MyClass<MyDefault>
````

Outlined in more detail [here](https://github.com/HaxeFoundation/haxe-evolution/pull/50#issuecomment-413976704)

## Detailed design

- Parse the new type parameter syntax for type declarations
- Ensure the default unifies with possible type guards
- Disallow a type parameter with a default to be followed by one without
- Use the default parameter when the type is used and there's none declared
- Other generic parameters can be used as long as they were defined before the default
  ````
  This should work: class A<B, C = B>
  This shouldn't: class A<B = C, C>
  ````
  The reasoning has been discussed in [other places](https://github.com/Microsoft/TypeScript/issues/2175) and works.

## Impact on existing code

If the defaults are available in macro context this can break existing macros 
that work with type parameters. Otherwise code that does not use the defaults
should function exactly the same.

## Drawbacks

- [Implicit types](https://github.com/HaxeFoundation/haxe-evolution/pull/50#issuecomment-418016806): It can cause some confusion because it's not easy to tell where a type came from.

## Alternatives

- It's possible to emulate with `@:genericBuild` but there's some downsides:
  - Can't use those in macros
  - Can't use the type as a type hint (`var a: MyGenericBuild;`)

- Parameters can be inferred on first usage, but only when constructing a type.

- Aliases can be used to set defaults: `typedef EmptyComponent = Component<{}, {}>`
It usually makes things more complex than necessary, see also: 
https://github.com/massiveinteractive/haxe-react/blob/19156680859ac0e27249762101cb8533b911a141/src/lib/react/ReactComponent.hx#L14.

## Unresolved questions

/
