# Command for empty project initialization

* Proposal: [HXP-NNNN](NNNN-init-command.md)
* Author: [Haxe Developer](https://github.com/dmitryhryppa)

## Introduction

Providinga a Haxe compiler command to create an empty project in the selected folder.

## Motivation

It is really annoying to create folders structure, files, declare a class and hxml params to just start coding. 
Also, you need to spend some time to setup VSCode and VSHaxe debuggers (which is an official editor, yes?).

This feature also will help a lot for new developers to start.

## Detailed design

To create a new empty project developer will need to call init command and optionally provide at least one target as a parameter:

`haxe --init cpp js hl`

Then Haxe will create a bunch of files and directories:
```
├───.vscode
│   ├── settings.json
│   └── tasks.json
├── out
│   ├── cpp
│   ├── hl
│   └── js
│       └── main.js
├── src
│   └── Main.hx
├── build.cpp.hxml
├── build.hl.hxml
├── build.js.hxml
├── common.hxml
```

`.vscode` with basic settings and tasks for launching and debugging (hxcpp, hl)

`src` with the Main.hx module and main class.

`out` with separated directories for each target

`common.hxml` with general compilation parameters. 

Bunch of `build.$TARGET.hxml` with compilation parameters for every target.



## Impact on existing code

Not impacting existing code.

## Drawbacks

None.

## Alternatives

None

## Unresolved questions

None.