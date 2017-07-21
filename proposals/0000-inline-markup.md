# Inline Markup

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Juraj Kirchheim](https://github.com/back2dos)

## Introduction

This is an attempt to salvage the rejected [Inline XML proposal](https://github.com/HaxeFoundation/haxe-evolution/pull/12).
The idea is to allow the compiler to identify "inline markup literals" as *opaque* expressions that can be processed by macros, with "markup" actually being as vague as possible. To illustrate what consitutes such inline markup:

1. For those who want JSX:

 ```haxe
 funtion doThatReactThing() 
   return <div class="container">
     <div class={whatever}>
       <a href={someUrl}>Click me!</a>
     </div>
   </div>
 ```
2. HEREDOC-like syntax has been requested for a long time now, and inline markup would allow to embed arbitrarily complex strings into haxe:

 ```haxe
 var myComplexString = 
   <SomeDelimiter>
     In here I can write "anything" without the parser really interfering
   </SomeDelimiter>
 ```
3. Or you know, just go crazy:

 ```haxe
 function madness()
   <JavaScript>
     console.log("This is Spartaaaaaaa!!!");
   </JavaScript>
 ```

4. And while you're at it, mix it any way you want it:

 ```haxe
 function someHTML() 
   return <html lang=en>
     <head>
       <title>Example</title>
       <meta name="example" content="This is just to point out that void elements are fine">
     </head>
     <body>
       <style>
         body {
           background: #FF00FF;/* everybody loves this color */
         }
       </style>
       <h1>Hello, world!<!-- hoho!!! --></h1>
       <div contenteditable>Edit me!</div>
       <script>
         alert("You're joking, right?");
       </script>
     </body>
   </html>
 ```
 
In each of those cases, Haxe is blisfully unaware of any language carnage going on *within* inline markup. It knows only enough to identify runs of inline markup, so that macros can pick them up and parse them. If an inline markup literal is not processed by any macro and makes it all the way to the typer, an error is generated (could just be "Unprocessed inline markup").

## Motivation

The main motivation here is really to make Haxe more appealing to web and desktop/mobile application developers. For better or worse, XML-ish dialects have establised themselves as the de-facto lingua franca of describing UIs for both web and native environments. XIB/Storyboard, XAML, MXML, FXML, JSX, HTML, HaxeUI markup, Stablex markup, ... you name it. If we want Haxe to gain traction outside game development, working on tighter integration between UI markup and the rest of the language is not a "nice to have" but of pivotal importance. 

That said this proposal aims to be a basis that opens many possibilities instead of comitting Haxe to support any single hype, thus giving macro authors the freedom of choosing whichever sinking ship they want to put their bets on.

## Detailed design

If wherever an expression is expected to begin, the character `<` is found followed *directly* (i.e. no whitespace inbetween) by a letter, it signifies the start of an inline markup expressions. Then the opening "tag" is determined (this will be something of the form `<foo.Bar_barf3-gnieh:blargh`) and we read forward until we find a matching closing "tag" (which would then be `</foo.Bar_barf3-gnieh:blargh>`), while continuing the search if that tag was not ballanced. Alternatively, if there is no `<` and `>` between the opening tag and the next `/>`, then we extract that text run as a markup literal. In Haxe code:

```haxe
var start = parser.pos;//which is at this point the position of the left angular bracket preceeding the letter
var parseTagName = ~/^[-._:a-zA-Z0-9]+/;
parseTagName.matchSub(source, start + 1);
var tagName = parseTagName.matched(0);
var openingTag = '<$tagName',
    closingTag = '</$tagName>';

parser.pos = start + openingTag.length;

//below `hasNeither` and `count` are left as an excercise to the reader

switch source.indexOf('/>', parser.pos) {
  case v if (v != -1 && hasNeither(source, parser.pos, v, ['<', '>'])):
    parser.pos = v + 2;
  default:
    do {
      switch source.indexOf(closingTag, parser.pos) {
        case -1: throw 'unclosed $openingTag';
        case v: parser.pos = v + closingTag.length;
      }
    } while (count(openingTag, source, start, parser.pos) != count(closingTag, source, start, pos)); //
}

return makeInlineMarkupExpression(source, start, parser.pos);//just turn it into some opaque expression
```

The result here would either be an `EConst(CMarkup(theMarkup))` or a `EMeta(':markup', EConst(CString(theMarkup)))` if we wish to avoid breaking changes to the AST. To repeat the above: **unprocessed markup literals should be rejected by the typer**. 

It has been suggested to have a default meaning, but I maintain for now that this is practically impossible to find one that will work well for all use cases and even if it is possible, it can be put forward in a follow up proposal that can be implemented at a later time.

## Impact on existing code

This feature does not impact existing non-macro code directly. Depending on how it is encoded as an expresion, it may break macros.

## Drawbacks

1. the proposal may be somewhat modest, because it still means that people have to write macros to process the inline markup. 
2. cases can be constructed with CDATA and comments that violate the principle of the least surprise:

    ```haxe
    var xml = <foo><![CDATA[</foo>]]></foo>;//will result in "expected ]"
    var xml = <foo><!-- </foo> --></foo>;//will result in "unexpected >"
    ```
    I'm going to go with: true, these cases exist, but they don't matter. There are countless ways to deal with the issue should it arise.

## Alternatives

1. Parse external files at macro time. This is pretty neat, but one if the biggest problems here is that you just can't convince the compiler to give you autocompletion. And you can't blame it, because it would have to type the whole codebase to figure out which macro is actually causing the file to be parsed and call it accordingly. 

2. Parse inline strings. This works *amazingly well* in `tink_hxx` (I am biased here, but I think it really shows the benefits of a proper, tight integration):

   ![](https://raw.githubusercontent.com/back2dos/js-virtual-dom/master/hxx-example.gif)
   
   However trying to get the parser to be reentrant (so you can write e.g. `<div>{[1,2,3].map(i -> <button>{i}</button>)}</div>`), requires you to parse Haxe syntax by hand, which not only is slow and desparate and silly, but also kills autocompletion ... unless you want to implement that part too, which is way beyond desparate and silly.
   
3. The idea was proposed to make such a language extension rely on compiler plugins. I'm not even sure it is possible to swap out the parser in this manner, but I would argue it is undesireable, because it makes all libraries that rely on custom parsers mutually exclusive. This is not so with macros, because macros are called explicitly through specific entry points and can therefore coexist in the same build.

## Opening possibilities

It has been pointed out that [Haxe could greatly benefit from an easy way to embed *any* (domain specific) language](https://github.com/HaxeFoundation/haxe-evolution/pull/12#issuecomment-306733033). This proposal happens to do that. It is fair to ask whether the surrounding delimiters need to be "tags". They don't: it could be any ASCII-art. However using tags has two advantages:

1. it just so happens that most people are used to tags acting as delimiters, so we don't have to reinvent the wheel here
2. if the embedded language is XML-ish, then the delimiter fuses with the language, which reduces visual clutter

## Unresolved questions

None so far.
