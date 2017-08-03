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

With the addition of a new Macro API the tag could be registered inside the compiler triggering an user function everytime the registered tag appear. For example, it could be an initialization macro like `--macro registerCodeBlock("xml", Macros.parseXmlCodeBlock)` with `Macros.parseXmlCodeBlock` being a static function that will accept the `ECodeBlock` AST node as a parameter and will return a new transformed expression. The compiler would call this function everytime a code block starts with ` ```xml ... ``` `


In addition, if the IDE supports it, it will be possible to highlight the block thanks to that tag.

## Detailed design

The behavior of this feature will be different if there is a tag or not.

- A simple block without a tag. In this case, the block would be parsed as if it were a single quote string block. The optional tag in the AST node would be `null`:

![http://i.imgur.com/2rOPnQE.png](http://i.imgur.com/2rOPnQE.png)

- A block with a tag. In this case, the block would be still parsed as a single quote string block but the optional tag in the AST node would be filled. If the tag is registered within the compiler the expression would be transformed by the registered function.


```
hxml file:
...
--macro registerCodeBlock("xml", Macros.processXmlBlock)
...
```

![http://i.imgur.com/8YwmRTa.png](http://i.imgur.com/8YwmRTa.png)

## Impact on existing code

It doesn't break existing code because it uses a new token that can't be used right now.

It modifies the AST nodes adding an extra node `ECodeBlock(original:String, ?tag:String, parts:Array<FormatFragment>)`

It adds a new Macro API to register the tag processing inside the compiler.

## Drawbacks

One drawback would be if the user registers multiple functions for the same tag. The compiler should emit an error if this happens. Another solution would be to limit the packages affected by the registered tag with a third parameter for it inside the `registerCodeBLock` macro function.

## Alternatives

There are multiple alternatives for this functionality that wouldn't change anything inside the compiler or add new AST nodes:

- Using an user macro with a metadata: `@:codeblock("jsx") var jsx = '...'; `
- Using an user macro with a function: `var jsx = jsx('...');` `jsx()` being a macro

While these options are known to work I feel that they aren't as recognizable for the user as the proposed pattern. They won't help the IDE with highlighting because the compiler won't be able to offer any help being these alternatives user made and not part of the language. The proposed pattern also avoids the need to escape `"` or `'` if the user wants to use it inside the string.

## Unresolved questions

I don't really like the proposal's or the AST node's names but I can't come with a better naming.
