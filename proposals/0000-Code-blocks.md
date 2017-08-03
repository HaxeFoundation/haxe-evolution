# Code blocks

* Proposal: [HXP-0000](0000-code-blocks.md)
* Author: [Justo Delgado Baudí](https://github.com/mrcdk)

## Introduction

Add support to code blocks with triple backticks for opening an closing them and an optional tag after the opening backticks ` ```tag ... ``` `

## Motivation

This proposal comes as an alternative to https://github.com/HaxeFoundation/haxe-evolution/pull/12 but leaving the block parsing and transformation to a user macro making it easier to the compiler to parse the block. 

This syntax for this proposal is inspired by the Markdown triple backtick code block syntax. If this syntax is to be followed, the tag may be optional making the code block behavior equal to a single quote string if there is no tag.

For the compiler, parsing the block would be the same as parsing a single quote string block with the addition of an optional tag after the starting three backticks ` ```tag something ${myObject.size} ``` ` would be the same as ` 'something ${myObject.size}' `.

It could use a similar AST node as this pull request adds https://github.com/HaxeFoundation/haxe/pull/6475 with the addition of an optional tag: `ECodeBlock(original:String, ?tag:String, parts:Array<FormatFragment>)`

It will not only add inline XML, as the mentioned proposal wanted, but add anything the user want: html, xml, jsx, a custom DSL,... 

In addition, if the IDE supports it, it will be possible to highlight the block thanks to that tag.

## Detailed design

The behavior of this feature will be different if there is a tag or not.

- A simple block without a tag. In this case, the block would be parsed as if it were a single quote string block. The optional tag in the AST node would be `null`:

![http://i.imgur.com/2rOPnQE.png](http://i.imgur.com/2rOPnQE.png)

- A block with a tag. In this case, the block would be still parsed as a single quote string block but the optional tag in the AST node would be filled:

![http://i.imgur.com/MXwU6Rp.png](http://i.imgur.com/MXwU6Rp.png)

## Impact on existing code

It doesn't break existing code because it uses a new token that can't be used right now.
It modifies the AST nodes adding an extra node `ECodeBlock(original:String, ?tag:String, parts:Array<FormatFragment>)`

## Drawbacks

Describe the drawbacks of the proposed design worth consideration. This doesn't include
breaking changes, since that's described in the previous section.

## Alternatives

I firstly thought about the triple single quote `'''` to start and finish a block like in Python but I think it would make parsing the haxe code more difficult so I chose the triple backtick ` ``` ` as in Markdown because I think it will be easier to parse and the user won't be confused if they see ` ```xml`

## Unresolved questions

I don't really like the proposal name or the AST node name but I can't come with a better naming.

My intial consideration was to add a new Macro API for the processing of the tag but I'm not sure how to do this. It could be an initialization macro like `--macro registerCodeBlock("xml", Macros.parseXmlCodeBlock)` with `Macros.parseXmlCodeBlock` being a static function that will accept the `ECodeBlock` AST node as a parameter and will return the new expression. The compiler would call this function everytime a code block starts with ` ```xml ... ``` `
