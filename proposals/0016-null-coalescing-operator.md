# Null coalescing operator

* Proposal: [HXP-0016](0016-null-coalescing-operator.md)
* Author: [RblSb](https://github.com/RblSb)
* Status: [to be implemented](https://github.com/HaxeFoundation/haxe/issues/10478)

## Introduction

Provide syntax for null coalescing operator, which returns the left operand if it is not null, and otherwise returns the right operand.

## Motivation

This operator exists in many other languages and makes null checks simplier and more readable. I think this would be nice addiction to Haxe syntax and future null-safe traversal operator.

## Detailed design

Example with usage of this operator, when `options` object is read-only:
```haxe
public static function create(options:Options) {
  final bgColor = options.bgColor ?? 0x000000;
  final title = options.title ?? "Cool Game";
  setOptions(bgColor, title);

  setSize(options.size ?? {w: 20, h: 10});
}
```
This would be same as:
```haxe
public static function create(options:Options) {
  final bgColor = options.bgColor == null ? 0x000000 : options.bgColor;
  final title = options.title == null ? "Cool Game" : options.title;
  setOptions(bgColor, title);

  setSize(options.size == null ? {w: 20, h: 10} : options.size);
}
```

Another possible usage with future null-safe traversal operator, if it will be implemented:
```haxe
return options?.advanced?.id ?? 0;
```

Return type of null coalescing operator will be non-nullable, same as with conditional operator. Incorrect type of right operand should error as before. In case of usage with multiple times operands are executed in left to right order, if no brackets are used:
```haxe
trace(foo ?? bar ?? 5);
// is ((foo ?? bar) ?? 5)
```

## Impact on existing code

Should be no impact.

## Alternatives

`foo ?: bar` syntax for this feature, that is used in some languages instead. But in this case would be harder to detect ternary operator typos.

## Opening possibilities

## Unresolved questions
