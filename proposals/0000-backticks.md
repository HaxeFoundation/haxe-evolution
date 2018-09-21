# Backtick support

* Proposal: [HXP-0000](0000-backticks.md)
* Author: [Mark Knol](https://github.com/markknol)

## Introduction

Support backticks <code>&#96;</code> in Haxe, which also support String Interpolation.

## Motivation

Escaping quotes in strings can be confusing, but it can be less if Haxe would support backticks.

```haxe
// best case at the moment
var html = '<button onclick="alert(\'Hi!\')">Press me</button>';
// or
var html = "<button onclick="alert(\"Hi!\")">Press me</button>";
//  worse case
var html = "<button onclick=\"alert(\\\"Hi!\\\")\">Press me</button>";
```
With backticks, everything can look nicer.
```haxe
var html = `<button onclick="alert('Hi!')">Press me</button>`;
```

## Detailed design

I'm not a compiler developer, but this is how I expect it to work.

- A backtick translates to `"`. 
  * <code>&#96;Backtick&#96;</code> translates to `"Backtick"`.
  * There is no direct need in Haxe/JavaScript to translate to ES6 backticks, since it's behavior is also slightly different.
- Single-quote `'` with nested backticks don't support interpolation, since this would be a breaking change.
  * <code>'Back &#96;tick&#96;'</code>   
    translates to  
    <code>&#39;Back &#96;tick&#96;&#39;</code>.
- Backticks use string interpolation. 
  * <code>var x=5; trace(&#96;x=${x}&#96;);</code>  
    translates to  
    `var x=5; trace("x=" + x + "");`
  * <code>var x=5; trace(&#96;x=$${x}&#96;);</code>  
    translates to  
    `var x=5; trace("x=${x}");` (no interpolation)
- Backticks with nested single-quote strings `'` support (nested) interpolation.
  * <code>var x=5,y=10; trace(&#96;Result=${if (true) 'x=${x}' else 'y=${y}'}&#96;);</code>  
    translates to  
    `var x=5,y=10; trace("Result=" + (true ? "x="+x : "y="+y));`.
  * You cannot infinitly nest because single-quote `'` with nested backticks don't support interpolation.

## Impact on existing code

There should be no impact on existing code since this is new syntax and doesn't affect current strings.

## Drawbacks

- If backticks are supported and have string interpolation, it might be confusing for (new) people that single-quote strings `'` also support string interpolation and `"` doesn't.
- Can't paste it into markdown :)
- Probably requires additional IDE support

## Alternatives

Not aware of any

## Opening possibilities

- It makes some ES6 code more copy/pastable to Haxe.
- Since it has one extra layer of string interpolation, it _could_ help with the jsx/xml kind of syntax people want. I see it being useful for the [hxx](https://github.com/haxetink/tink_hxx) library for example. 

## Unresolved questions

- How should it be represented in AST
- Does this introduce issues for unicode support
- Macro encoding might be an issue, because it's already an issue with `"` vs. `'`
