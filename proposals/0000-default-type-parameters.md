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

See also: https://github.com/HaxeFlixel/flixel/issues/1677

## Detailed design

- Parse the new type parameter syntax for type declarations
- Ensure the default unifies with possible type guards
- Disallow a type parameter with a default to be followed by one without
- Use the default parameter when the type is used and there's none declared

## Impact on existing code

If the defaults are available in macro context this can break existing macros 
that work with type parameters. Otherwise code that does not use the defaults
should function exactly the same.

## Drawbacks

Don't see any at this time.

## Alternatives

- It's possible to emulate with `@:genericBuild` but there's some downsides:
  - Can't use those in macros
  - Can't use the type as a type hint (`var a: MyGenericBuild;`)

- Parameters can be inferred on first usage, but only when constructing a type.

- Aliases can be used to set defaults: `typedef EmptyComponent = Component<{}, {}>`
It usually makes things more complex than necessary, see also: 
https://github.com/massiveinteractive/haxe-react/blob/19156680859ac0e27249762101cb8533b911a141/src/lib/react/ReactComponent.hx#L14.

## Unresolved questions

- Should the defaults have access to the other parameters?
  `Test<A, B, C = A & B>` This would introduce a lot of edge cases.
