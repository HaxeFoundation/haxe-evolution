# Enum abstract instances

* Proposal: [HXP-NNNN](0000-enum-abstract-instances.md)
* Author: [markknol](https://github.com/markknol)

## Introduction

Allow instances as enum abstract values. 

```haxe
@:forward
enum abstract Person( { final name:String; final age:Int; } )  {
  final Mark = { name: "Mark", age: 21 };
  final John = { name: "John", age: 25 };
  final Elisa = { name: "Elisa", age: 28 };
}
```
This would allow to use the enum be used in switches too:
```haxe
final person:Person = Mark;

var message = switch person {
  case Mark: 'Hello from The Netherlands, my name is ${person.name}';
  case John: 'Hello from the USA, I am ${person.age} years old';
  case Elisa: 'Hello from Japan';
}

trace('Message: $message');
```

## Motivation

Currently when you try to use instance values in enum abstract the compiler gives an error:  
`Inline variable initialization must be a constant value`

This can lead to using workarounds that's more error-phrone. Allowing instances could be a pragmatic and neat solution.

Allowing instances complies and compliments the goals of enum abstracts:

* The typer can ensure that all values of the set are typed correctly.
* The pattern matcher checks for exhaustiveness when matching an enum abstract.
* Defining fields requires less syntax.

## Detailed design

* All values are generated as static read-only singletons. 
* Values are required. In case primitive types sometimes values are generated.
* Current enum abstract values are inlined, but that shouldn't happen for instances. 
* When using the values in a `switch` (pattern matching), an `if-else` branch should be generated `if (person == Person.Mark) ..`.

## Impact on existing code

There should be no impact on existing code, since this is a new feature, a removal of a current restriction.

## Drawbacks

One argument against this is that we get these hidden if-else chains when pattern matching. But then again that's already the case for String matching on most targets (Although that can in theory be optimized more)

## Alternatives

The alternatives are always workarounds where you lose the type-safety or the nice things that Haxe pattern matching brings.  
For example you can create a static map to the enum abstract with a `.get()` function, but that is slower but also takes more time to write. But this never guarantees that all values are in.  
Another alternative would be to have a Pair type. This also doesn't guarantees that all values are in.  Both also don't give the power of pattern matching.

## Opening possibilities

A nice illustration of the usage/possibilities would be an extended version of the "Taste of haxe" (from haxe.org).

```haxe
class Game {
  // Haxe applications have a static entry point called main
  static function main() {
    // Anonymous structures.
    var playerA:Player = { person: John, move: Paper }
    var playerB:Player = { person: Jane, move: Rock }
        
    // Array pattern matching. A switch can return a value.
    var result = switch [playerA.move, playerB.move] {
      case [Rock, Scissors] | 
           [Paper, Rock] |
           [Scissors, Paper]: Winner(playerA);
            
      case [Rock, Paper] |
           [Paper, Scissors] |
           [Scissors, Rock]: Winner(playerB);
            
      case _: Draw;
    }
    // Paper vs Rock, who wins?
    var message = switch result {
      case Result(player): 
        switch player.person { 
          case John: 
            'Our ${player.person.name} wins a lot, even though he is ${player.person.age} years old!';
          case Jane | Jane: 
            'One of the women won, it was ${player.person.name}!';
          case other: 
            'An interesting round, but ${other.person.name} has won this time!';
        }
      case Draw: 
        "Draw!";
    }
    // log the message
    trace(message);
  }
}

enum abstract Person({ final name: String; final age:Int }) {
  final John = { name: "John Bon", age: 65 };
  final Jane = { name: "Jane Dane", age: 18 };
	final Mark = { name: "Mark Hark", age: 36 };
	final Elisa = { name: "Elisa Disa", age: 42 };
}

// Allow anonymous structure named as type.
typedef Player = { 
  final person: Person; 
  final move: Move;
}

// Define multiple enum values.
enum Move { Rock; Paper; Scissors; }

// Enums in Haxe are algebraic data type (ADT), so they can hold data.
enum Result { 
  Winner(player:Player); 
  Draw; 
}
```

## Unresolved questions

- Would it be bad / confusing that you can alter values inside the instance?
- There a concern for hidden cost of this feature? 
- How can this be effectively be generated for each platform?
