# Feature name

* Proposal: [HXP-NNNN](NNNN-inline-xml.md)
* Author: [Kevin Leung](https://github.com/kevinresol)

## Introduction

Write XML within Haxe code directly. Example:

```haxe
var myXml = <elem>value</elem>;
```

## Motivation

One can write inline XML/HTML just like jsx, which is a fruitful way to represent hierachical structures.

jsx is a good example in making use of inline XML.

## Detailed design

Since `<` itself is current a binary operator which requires a operand on its left hand side, 
so when there is no expression on its left hand side we can consider it as the start of a inline XML declaration.
Alternatively, we can reference jsx's way of doing it, that is using a parenthesis+lessthan sequence `(<` to declare
the start of the XML.

In macro, we need to add one more ExprDef: `EXml(xml:Xml)` 
(I am not very sure about the contents here, maybe we need some sub-nodes like 
`XmlElement(name:String, attr:Array<XmlAttribute>, children:...)`, `XmlAttribute(name:String, value:String)`, etc)

Interpolation should also be supported, we can use the same dollar sign as in string interpolation 

```haxe
var v = "some value";
var xml = <div>$v</div>
var wrapper = <html>$xml</html>
```

## Impact on existing code

This is a breaking change because new node is introduced in ExprDef.

## Drawbacks

Requires additional support from IDE (and completion server), in terms of both code completion and syntax highlighting.

## Alternatives

In haxe-react, effort is made to parse xml (in string format) at compile-time with macro. But there are some limitations like:

- string escaping, there are some extra quote-escaping because one pair of quote is used to quote the xml string itself
- cannot nest xmls in interpolation
  
  For example, in jsx, the following is possible
  ```js
  var i = someValue();
  return <div>{i == 0 ? <span>nothing</span> : <span>We have {i} items</span>}</div>
  ```
  But in haxe's macro-based solution, the following obviously won't work
  ```haxe
  var i = someValue();
  return jsx('<div>{i == 0 ? jsx('<span>nothing</span>') : jsx('<span>We have {i} items</span>')</div>');
  ```
- compiler flag not supported, because the xml string is passed in as a whole and you can't split it (because it is one single Expr at compile time)

## Unresolved questions

(AS3 supports inline XML I wonder why Haxe doesn't, maybe there are some reasons behind it and I am willing to learn)
