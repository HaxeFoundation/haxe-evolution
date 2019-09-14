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

#### 1a. Comment Enum

An enum would be created to represent different styles of comments:

```haxe
enum Comment
{
    // example (starts with '//')
    Single(value:String); 
    
    /*
	example  (starts with '/*')
	*/
    Multi(value:String); 
    
    /**
    * Javadoc style example (starts with '/**')
    */
    Doc(value:String)
}
```

This would also leave room for more styles of comments to be added over time (e.g. Markdown, HTML).

##### 1b.  Improved Javadoc parsing

Currently when Javadoc style comments get parsed, the leading `*` on each line isn't stripped. Haxe documentation doesn't use these on each line, but most languages/developers do (and many IDEs add this formatting by default).

This means that code generators that never use this syntax (e.g. python) have to strip this leading character off, or put up with poorly formatted comments.

The proposal here is to assess the comment block and strip the minimum amount of whitespace with an optional `*` char that occurs on **all** lines of the comment.

#### 2. Types and Fields

These could be accommodated by adding a `?comments:Array<Comment>` prop to the `BaseType`, `ClassField` & `Field`.

```haxe
/* Some Class Information */
class BundleAnalytics
```

This does mean that comments trailing class definitions would be omitted.

#### 3. Parsing Javadoc more

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
    Doc(value:DocBlock)
}
typedef DocBlock =
{
    ?description: String,
    ?version: String,
    ?returns: String,
    ?authors: Array<String>
    ?params: Array<{name:String, doc:String}>,
}
// Or something more simple
typedef DocBlock =
{
    ?description: String,
    ?params:Array<{name:String, doc:String}>,
    ?tags:Array<{name:String, value:String}>,
}
```

#### 4. Code Generation

When using the compiler, specifying `-Dcomments` would request that the generated code contains comments.

It may also be useful to have a few options on this:

```
-Dcomments=all // include all comments
-Dcomments=type // only include comments on classes
-Dcomments=type+field // only include comments on classes and fields
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

None.

## Alternatives

Currently Haxe does have a simple implementation of comments in classes and fields (`doc` property), but this support has several limitations.

- Only retains Javadoc style comments, no single or multi-line comments.
- Doesn't support multiple comments per entity.
- Doesn't cope well with Javadoc style comments (i.e. retains leading whitespace and '*' on each line).
- Doesn't have a way to include these comments in the generated code (as far as I know).

My feeling is that `doc` should be retained for backwards compatibility for a while before being removed in v5.

## Opening possibilities

This could open up Haxe for using comments as containers for certain DSLs.

## Unresolved questions

- What level of control to give with compiler flag (`-Dcomments`)
- Is using an enum overkill? Could also just be a single typedef with optional fields.
- Is keeping an array of comments against every entity overkill? Maybe a single Comment would suffice.