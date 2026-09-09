+++
title = "Mixing instances of parent and child in Crystal"
date = "2023-12-14"
tags = ["crystal", "lucky"]
categories = ["general"]
authors = ["mike"]
description = "A strange error message from the Crystal compiler."
+++
I've been getting into [Crystal](https://crystal-lang.org) lately.  Its syntax is a near-copy of [Ruby's](https://ruby-lang.org/), but its type system and compile-time checks are extremely strong.  It compiles using [LLVM](https://llvm.org/), like [Rust](http://rust-lang.org) does, which results in really fast binaries.

While the language is fairly readable, its error-messages aren't always clear. I was working on something like this today:

```crystal
class Animal
  def initialize
  end
end

class Dog < Animal
  def initialize(name : String)
  end
end

class PetCemetary
  def initialize(@pets : Array(Animal | Dog))
  end
end

rufus = Dog.new(name: "rufus")

deadPets = Array(Animal).new
deadPets << rufus
cemetary = PetCemetary.new(deadPets)
```

This yielded a compile-error:

```sh
Showing last frame. Use --error-trace for full trace.

In classTest.cr:20:28

 20 | cemetary = PetCemetary.new(deadPets)
                                 ^-------
Error: expected argument #1 to 'PetCemetary.new' to be Array(Animal), not Array(Animal)

Overloads are:
 - PetCemetary.new(pets : Array(Animal | Dog))
```

In particular:
> Error: expected argument #1 to 'PetCemetary.new' to be Array(Animal), not Array(Animal)

The error message implies a type-mismatch, but the types are identical.

So what's the issue?

Crystal can accept descendant classes (in this case, `Dog`) where they just ask for the parent classes (here that's `Animal`).  So the solution was to remove the union from the parameter, and the `Dog` instance will still be accepted in to an array of `Animal`:

```crystal
class Animal
  def initialize
  end
end

class Dog < Animal
  def initialize(name : String)
  end
end

class PetCemetary
  def initialize(@pets : Array(Animal))
  end
end

rufus = Dog.new(name: "rufus")

deadPets = Array(Animal).new
deadPets << rufus
cemetary = PetCemetary.new(deadPets)
```

But why would union of its type and sub-type throw an error?  Especially if its subtype and parent type are interchangeable.  Sure, the union is redundant, but not erroneous.

After discussing the error on the Crystal discord (thanks to **@Blacksmoke16** and **[@HertzDevil](https://github.com/HertzDevil)** for checking this out) it was determined this is a compiler bug of sorts. Not a terrible one, but confusing nonetheless.

Basically, the type-checker might be [overly restrictive.](https://github.com/crystal-lang/crystal/issues/14091) when comparing types in a union operator.  Things are a little more complicated by the fact that while it throws an error when expressing things in a `ParentType | ChildType` format, it doesn't throw an error when expressing it as `Union(ParentType, ChildType)`.  So really, its an inconsistency.

Anyways, I stil think Crystal is a fantastic language, and despite sometimes cryptic compile-time errors, I find the language is easy to use, productive, and relatively safe.  If you like the Ruby's syntax, but can't stand its sluggish performance, not to mention lack of null/nil safety and duck-typing system, give Crystal a shot. What's more, I've found the community of developers and contributors really friendly and responsive.
