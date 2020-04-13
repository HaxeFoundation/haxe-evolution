# ::Macro instances::

* Proposal: [HXP-NNNN](NNNN-filename.md)
* Author: [Michal Romecki](https://github.com/filt3rek)

## Introduction

Having macro instances in expressions macro context.

## Motivation

In expressions macro context, all methods are statics so we can't have macro instances variables.

This proposal comes from a simple macro function that checks in a template HTML file if an tag id exists. When in a regular class you have many ids to check, in many places in your code, you have always to put the HTML template path in order to retrieve from a static var the parsed ids for example.

For example :
```haxe
class HtmlFile{
	static var hids		: Map<String,Array<String>> = [];
	macro public static function getElementById( path : String, id : String ){
		if( !hids.exists( path ) ){
			// Load and parse file
			// Store ids in an array
			// Store this array in "hids" static hash map
		}
		var ids	= hids.get( path );
		if( ids.indexOf( id ) == -1 ){
			Context.fatalError( 'id "$id" doesn\'t exist in $path', Context.currentPos() );
			return null;
		}else{
			return macro js.Browser.document.getElementById( $v{ id } );
		}
	}
}
```
And then in regular classes, use it like that :
```haxe
var myElement1	= HtmlFile.getElementById( "path/to/my/template.mtt", "myId1" );
var myElement2	= HtmlFile.getElementById( "path/to/my/template.mtt", "myId2" );
var myElement3	= HtmlFile.getElementById( "path/to/my/template.mtt", "myId3" );
```

It would be better if we could create a macro instance with the path and then just call a member method with the id to retrieve like that :

```haxe
class HtmlFile{
	var ids		: Array<String> = [];
	macro public function new( path : String ){
		// Load and parse file
		// Store ids in "ids" array
	}

	macro public function getElementById( id : String ){
		if( ids.indexOf( id ) == -1 ){
			Context.fatalError( 'id "$id" doesn\'t exist in $path', Context.currentPos() );
			return null;
		}else{
			return macro js.Browser.document.getElementById( $v{ id } );
		}
}
```
And then in regular classes, use it like that :
```haxe
var myHtmlFile	= new HtmlFile( "path/to/my/template.mtt" );
var myElement1	= myHtmlFile.getElementById( "myId1" );
var myElement2	= myHtmlFile.getElementById( "myId2" );
var myElement3	= myHtmlFile.getElementById( "myId3" );
```

Of course, it's nothing blocking, we can still use statics and do with that and I'm sure I'm not inventing something amazing but there are many other cases where this "feature" can be useful.

## Detailed design

I have no idea how this can be implemented but according to [this discussion](https://community.haxe.org/t/macro-instance/2347/11) it seems it can technically be done, it miss "just" a syntax ?

## Impact on existing code

It seems it doesn't break anything since it "just" needs an additional syntax ?

## Drawbacks

No drawback ?

## Alternatives

Many...

## Unresolved questions

Find a syntax to interact with it ?
