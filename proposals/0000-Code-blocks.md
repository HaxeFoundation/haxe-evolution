# Code blocks

* Proposal: [HXP-0000](0000-code-blocks.md)
* Author: [Justo Delgado Baudí](https://github.com/mrcdk)

## Introduction

This proposal comes as an alternative to https://github.com/HaxeFoundation/haxe-evolution/pull/12 but leaving the block parsing and transformation to an user macro. It's achieved by the use of triple backticks ` ``` ` as opening and closing tokens and an optional tag after the opening backticks ` ```tag ... ``` ` that would apply an user macro to the block.

## Motivation

As the introduction said, this proposal is an alternative to add inline XML. While said proposal wanted to add a fully XML parser to the compiler, this proposal leaves that functionality to the user via a macro function and doesn't stop with XML-like dialects like XML, HTML, JSX,... but let anything to be embedded.

One could argue that, if the sole purpose of this proposal is to add yet another way to call a macro function, why even bother adding this feature. My reasoning for the addition of this feature is clear: Make the compiler aware about the content of the block, something that wouldn't be possible with a macro. Making the compiler aware of this would help other tools like, for example, an IDE can use this information for, if supported, add highlighting, auto completion,... to that specific block. Another example would be the usage of an external linter inside that block.

## Detailed design

This syntax for this proposal is inspired by the Markdown triple backtick code block syntax. If this syntax is to be followed, the tag may be optional making the code block behavior equal to a single quote string if there is no tag.

For the compiler, the content would be the same as if the block were surrounded by single quotes with the addition of an optional tag after the starting three backticks ` ```tag something ${myObject.size} ``` ` would be the same as ` 'something ${myObject.size}' ` with the extra step of passing that string to an user macro. 

It could use a similar AST node as this pull request suggests https://github.com/HaxeFoundation/haxe/pull/6475 with the addition of an optional tag parameter: `ECodeBlock(original:String, ?tag:String, parts:Array<FormatFragment>)`

With the addition of a new Macro API the tag could be registered inside the compiler triggering an user function everytime the registered tag appear. For example, it could be an initialization macro like `--macro registerCodeBlock("xml", Macros.parseXmlCodeBlock)` with `Macros.parseXmlCodeBlock` being a static function that will accept the `ECodeBlock` AST node as a parameter and will return a new transformed expression. The compiler would call this function everytime a code block starts with ` ```xml ... ``` `

The behavior of this feature will be different if there is a tag or not.

- A simple block without a tag. In this case, the block would be parsed as if it were a single quote string block. The optional tag in the AST node would be `null`:

![http://i.imgur.com/2rOPnQE.png](http://i.imgur.com/2rOPnQE.png)

- A block with a tag. In this case, the block would be still parsed as a single quote string block but the optional tag in the AST node would be filled. If the tag is registered within the compiler the expression will be transformed by the registered function.

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

But, as explained in the Motivation point, the compiler wouldn't be aware of what that string is representing and wouldn't be able to offer information to other tools. The proposed pattern also avoids the need to escape `"` or `'` if the user wants to use it inside the string.

There are other alternatives that use different opening and closing tokens. I chose ` ``` ` because I feel it would be easy for the parser to parse them and, because it's only found on Markdown, it will lower the chance to create issues if the closing token is present before the block ends.

On [this proposal](https://github.com/back2dos/haxe-evolution/blob/master/proposals/0000-inline-markup.md) by Juraj Kirchheim he proposes XML tags `<tag>...</tag>` as opening and closing tokens.

ES6 offers [template literals](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Template_literals) which are the same as Haxe single quote strings. But it also offers the possibility of tagging them like:
```
tag`my string`
```
The only problem I see with this is that people coming from JavaScript will infer that the processing of the block can be done at runtime which won't work with Haxe because string interpolation is a compile-time feature only.


## Unresolved questions

I don't really like the proposal's or the AST node's names but I can't come with a better naming.

Should the compiler emit a warning if the tag isn't registered? The tags may be only used to document the block of code.
