# Comments in AST

* Proposal: [HXP-0000](0000-comments-in-ast.md)
* Author: [Tom Byrne](https://github.com/TomByrne)

## Introduction

A more comprehensive representation of code comments in Haxe AST would make for much more readable output and facilitate people using Haxe as a source-to-source tool.

Using a compiler flag, comments could be included in the generated code (e.g. `-Dcomments`) .

This would replace the `doc` property that currently exists on `ClassType`, `ClassField` & `Field`.

## Motivation

As Haxe gets more usage in different domains, there is more emphasis on the generated code being human-readable. Retaining code comments in the generated code would dramatically improve readability. 

This would make Haxe more appealing as a source language for core business logic that is transpiled to multiple destination languages and used by multiple over teams (who may not be comfortable back-referencing the source Haxe code to find more information).

## Detailed design

#### Comment Enum

An enum would be created to represent different styles of comments:

```haxe
enum Comment
{
    // example
    Single(value:String); 
    
    /* example */
    Multi(value:String); 
    
    /**
    * Javadoc style example
    */
    Doc(value:String)
}
```

Parsing of Javadoc style comments would differ slightly from the current implementation in that the leading whitespace and `*` character of each line would be stripped, so that the content of the comment can be written to other languages that use different comment syntax (e.g. python).

#### Classes and Fields

These could be accommodated by adding a `?comments:Array<Comment>` prop to the `ClassType`, `ClassField` & `Field`.

```haxe
/* Some Class Information */
class BundleAnalytics
```

This does mean that comments trailing class definitions would be omitted.

#### Near Expressions

I'll detail a few different approaches to comments in expressions for discussion.

**Example:**

```haxe
while(true) /* comment 1 */
{
    // comment 2
    func();
    // comment 3
}
```

**Solution 1:**

Adding comments to the `Expr` typedef is a simple solution that catches most cases, e.g. `?comments:Array<Comment>`.

The above example might look something like this (position info removed for readability):

```haxe
{
    expr:EWhile($v{true},
                {
                    expr: EBlock{
                        [{
                            expr:ECall($i{'func'}, []),
                            comments:[Comment.Single("Comment 2")]
                        }]
                    },
                    comments:[Comment.Multi("Comment 1")]
                }, true),
}
```

Note: This solution doesn't have a nice place to put Comment 3, so the comment is omitted.

**Solution 1a:**

To cover more cases, another property `?postComments:Array<Comment>` could be added to `Expr` which could be used to catch any dangling comments that can't be pinned to a subsequent expression.

The downside is that this introduces a little ambiguity as to where a comment ends up; on the previous expression or the next one.

```haxe
{
    expr:EWhile($v{true},
                {
                    expr: EBlock{
                        [{
                            expr:ECall($i{'func'}, []),
                            comments:[Comment.Single("Comment 2")],
                            postComments:[Comment.Single("Comment 3")]
                        }]
                    },
                    comments:[Comment.Multi("Comment 1")]
                }, true),
}
```



**Solution 2:**

Adding comments as an enum to ExprDef is a simple solution that accounts for many cases.

```haxe
{
    expr:EWhile($v{true},
                {
                    expr: EBlock{
                        [{
                            expr:EComment(Comment.Single("Comment 2"))
                        },{
                            expr:ECall($i{'func'}, [])
                        },{
                            expr:EComment(Comment.Single("Comment 3"))
                        }]
                    }
                }, true),
}
```

Note: This solution doesn't have a nice place to put Comment 1, so the comment is omitted.

**Solution 2a:**

A combination of Solution 1 and 3 could be used, where comments are created as Exprs if possible, otherwise they're pinned to the following Expr:

```haxe
{
    expr:EWhile($v{true},
                {
                    expr: EBlock{
                        [{
                            expr:EComment(Comment.Single("Comment 2"))
                        },{
                            expr:ECall($i{'func'}, [])
                        },{
                            expr:EComment(Comment.Single("Comment 3"))
                        }]
                    },
                    comments:[Comment.Multi("Comment 1")]
                }, true),
}
```

The downside here is that it adds ambiguity about how comments should be exist in the AST, and is a bit more verbose.

#### Edge cases

That said, it may not be possible to support comments in every part of the code, as they lack a suitable structure to be pinned to in AST, for example:

```haxe
var myProp:Null<MyStrAbstract/*String*/>;
```

As comments are essentially 'sugar', I think it's ok that comments that don't fit nicely into the AST are discarded.

These situations will normally happen when comments are being used to contain old code, rather than useful information, so it may not be a 'real' issue.

#### Parsing Javadoc more

It may also be worth parsing Javadoc style comments in greater detail, so that they can be reformatted for the conventions of the target language.

For example, in python:

```python
"""Method description

:param param1: Readable description
:returns: formatted string
"""
```

In this case, the Comment.Doc enum would look something like this (more fields on [wikipedia](https://en.wikipedia.org/wiki/Javadoc)):

```haxe
enum Comment
{
    ...
    Doc(value:FieldDoc)
}
typedef FieldDoc =
{
    ?description: String,
    ?version: String,
    ?returns: String,
    ?authors: Array<String>
    ?params: Array<{name:String, doc:String}>,
}
```

#### Code Generation

When using the compiler, specifying `-Dcomments` would request that the generated code contains comments.

It may also be useful to have a few options on this:

```
-Dcomments=all // include all comments
-Dcomments=class // only include comments on classes
-Dcomments=class+field // only include comments on classes and fields
-Dcomments=expr // only include comments in expression blocks
-Dcomments // implies 'all'
```

Certain targets may not support single line comments in certain places, it would be up to the Code Generator whether to use a multiline comment or omit the comment altogether in this case.

As this is essentially sugar, so if support isn't straightforward in some circumstances, then comments should simply be omitted silently.

## Impact on existing code

Existing Haxe code would not be affected.

The Haxe parser would need to be updated first.

As this is extra functionality and is opt-in, I think it's ok to add this gradually to each of the targets, and throw a compiler error if it is used on an unsupported target in the meantime.

Some targets would never support this and would always throw an error (i.e. SWF).

## Drawbacks

None of the solutions offer a guarantee that every comment will make it to the target language.

## Alternatives

Currently Haxe does have a simple implementation of comments in classes and fields (`doc` property), but this support has several limitations.

- Only retains Javadoc style comments, no single or multi-line comments.
- Doesn't allow for comments within expressions.
- Doesn't support multiple comments per entity.
- Doesn't cope well with Javadoc style comments (i.e. retains leading whitespace and '*' on each line).
- Doesn't have a way to include these comments in the generated code (as far as I know).

My feeling is that `doc` should be retained for backwards compatibility for a while before being removed in v5.

## Opening possibilities

This could open up Haxe for using comments as containers for certain DSLs.

## Unresolved questions

- How to represent comments within expressions
- What level of control to give with compiler flag (`-Dcomments`)