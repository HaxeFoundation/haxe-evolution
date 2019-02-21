# Expression macro method forwarding

* Proposal: [HXP-0009](0009-macro-forward.md)
* Author: [Dan Korostelev](https://github.com/nadako)

## Introduction

Provide a way to have expression macro functions inside a class without typing that whole class
in macro context.

## Motivation

It's often desirable to have a macro function inside a "normal" class/abstract to achieve
nice API.

However, doing this will cause the module containing a macro function to be also typed
in the macro context, even if the macro function doesn't use anything from that module.

This in turn can lead to annoying and surprising errors, since the other code in that
module wasn't written with macros in mind and can depend on APIs that aren't even available
in macros (e.g. target-specific ones).

For example:
```haxe
class Main {
    static function main() {
        js.Browser.console.log("Version " + getVersion());
    }

    static macro function getVersion() {
        return MyMacroTools.getVersionExpr();
    }
}
```
This will fail with `You cannot access the js package while in a macro (for js.Browser)`
without even specifying the actual cause of the error.

And even if it does pass through the typer, most of the code will simply be
unused in macro context, so it's just a needless work consuming precious time.

The current solution is to fence the non-macro code (and imports) with `#if !macro`,
which hurts readability and often confuses people not familiar with macros, causing
them to "wtf" by accidentally adding a function outside of `#if !macro` region.

What I'd like to propose is a way to avoid the very cause of the issue by forwarding
macro calls into another module.

## Detailed design

The proposed solution is quite simple: allow macro methods without body if they have
a special metadata pointing to the static method in another type. For example, the function
from the example above could be defined like this instead:

```haxe
@:forwardMacro(MyMacroTools.getVersionExpr)
static macro function getVersion();
```

When typing a macro-call, if the macro method has no body, the compiler would look
for that metadata and extract the path to a real method to be evaluated. So if that
real method is in another module, the current one doesn't need to be typed in macro
context at all.

In addition to per-method macro we could have class-level `@:forwardMacro(MyMacroClass)`
metadata that will forward all bodyless macro methods from this class to the specified `MyMacroClass`.

## Impact on existing code

This should have no impact on existing code whatsoever, because it doesn't bring any
new syntax or change any existing behaviour.

## Drawbacks

One could argue that very simple macro methods should rather be in a class along
with the others, however in practice, macro code is never that simple and almost
always deserves to be separated. And anyway, it'll be still possible to do using
good old `#if !macro` fencing.

## Opening possibilities

While not really related, there's another macro-related proposal I'd like to prepare
in near future about having "macro modules", which should play well with the method
forwarding functionality.

