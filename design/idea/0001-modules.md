# [#0001] - Modules

|             |      |
| ----------- | ---- |
| **Authors** | Quil |

- [ ] Discussion
- [ ] Implementation

## Summary

As a modular programming language, Origami aims to support both _modularity_ and _modular compilation_. We also care a lot about allowing mutual recursion between modules (disallowing it causes people to organise their programs in awkward ways), and security. The module system also has to be implementable in JavaScript (our target language) with acceptable performance.

There are not many module systems in usage that fit all of the criteria above, at least not in widely used languages. Origami brings some ideas from academia and implements them. In short, the module system:

- Separates `modules` and `interfaces`, like ML does. Interfaces describe a type signature for what we expect of a module, and are identified by an unique name. Modules are anonymous implementations of these interfaces.

- Modules can only depend on interfaces. There's no global namespace, and free variables are not allowed in a module--so linking is explicitly speficied. Implementations are provided through the Linking phase, which supports the use case of things like dependency injection and parameterised modules naturally without the need for functors or containers.

- No executable form is allowed at the top-level of a module. A module is a declarative entity, and it should be safe to load it and analyse its contents in full without the risk of running some attacker code. This also enables mutually-recursive modules, as names are naturally late-bound.

- Linking takes in a constraint (what we want) and a search space (what we can link), and provides some module implementation. Ambiguity is not allowed in this phase, and the user has to disambiguate manually (by providing more constraints or limiting the search space). Search spaces are per-module in an hierarchical fashion to reduce the burden of defining them, but without reducing security properties too much.

## References

### Modules and modularity

- [A Ban on Imports](https://gbracha.blogspot.com/2009/06/ban-on-imports.html)  
  -- Gilad Bracha, 2009

- [A Ban on Imports (continued)](https://gbracha.blogspot.com/2009/07/ban-on-imports-continued.html)  
  -- Gilad Bracha, 2009

- [Modularity Without a Name](https://awelonblue.wordpress.com/2011/09/29/modularity-without-a-name/)  
  -- David Barbour, 2011

- [Modules Divided: Interface and Implement](https://awelonblue.wordpress.com/2011/10/03/modules-divided-interface-and-implement/)  
  -- David Barbour, 2011

- [Modules as Objects in Newspeak](http://bracha.org/newspeak-modules.pdf)  
  -- Gilad Bracha, Peter von der Ahé, Vassili Bykov, Yaron Kashai, William Madok, and Eliot Miranda, 2010

- [F-ing Modules](https://people.mpi-sws.org/~rossberg/f-ing/)  
  -- Andreas Rossberg, Claudio Russo, and Derek Dreyer, 2014

### Relavant implementations

- [Racket's Units](https://docs.racket-lang.org/guide/units.html)
- [OCaml's modules](https://ocaml.org/learn/tutorials/modules.html)
- [Haskell's Backpack](https://plv.mpi-sws.org/backpack/)
- [Newspeak's modules](http://newspeaklanguage.org/)

### Relational & Constraint logic

- [Relational Programming in miniKanren: Techniques, Applications, and Implementations](https://github.com/webyrd/dissertation-single-spaced/blob/master/thesis.pdf)  
  -- William E. Byrd's PhD thesis, 2009

- [µKanren: A Minimal Functional Core for Relational Programming](http://webyrd.net/scheme-2013/papers/HemannMuKanren2013.pdf)  
  -- Jason Hemann, and Daniel P. Friedman, 2013

- [cKanren: miniKanren with Constraints](http://www.schemeworkshop.org/2011/papers/Alvis2011.pdf)  
  -- Claire E. Alvis, Jeremiah J. Willcock, Kyle M. Carter, William E. Byrd, and Daniel P. Friedman, 2011

## Motivation

As programs grow more complex, programmers need to start organising them into more manageable pieces—components. As programs grow even more complex, programmers need to incorporate components written and maintained by other people.

The first of these tasks poses a modularity problem: how do you define components? How do you make these components stand on their own? How can you switch components by equivalent ones if needed? How can you extend components to do new things?

The second of these tasks also poses a security problem: can you trust code written by other people? How much can you trust it? Will adding this code break other parts of my application that have nothing to do with it? What happens if they change the code? How difficult is it to update components to fix a security problem? Can you temporarily fix a component without having to maintain it forever if you discover a huge security issue?

Sadly, most widely used programming languages do pretty poorly at both of these problems. Modules are more often than not entangled by name, path, or some similar identification, so it's difficult to switch one for another. Modules can use anything your program has access to, so you have to fully trust every piece of code you add to your program. And some languages have features which are not modular (such as Type Classes in Haskell), where modules may conflict in a way that render your application unusable.

Some work-arounds for these problems, like dependency injection containers in Java, and stack-inspection for security in the JVM, result in a lot of complexity--so applications are harder to write and debug,--and performance overhead.

Origami is a language that values safety--it's one of its core goals,--so any unsafe module system is unacceptable. We also value practicality, so we'd like to have a system that isn't complex to reason about or use, but that can't come at the expense of security and safety.

Luckily, people have been working on these issues for a long time, and while there isn't a language that coherently brings these together ([Pony](https://www.ponylang.io/) and [Newspeak](http://newspeaklanguage.org/) have promising security facilities, however), it's possible to combine different ideas into one module system that comes close to our ideal. That's what this document describes.

## An overview of the module system

Origami's module system is primarily based on David Barbour's idea of linking modules using constraints, and dividing modules into interfaces and anonymous implementations of interfaces. The latter is also similar to ML module systems to an extent--you have a module signature that defines the expected shape for a module, and module implementations which provide concrete definitions of these signatures.

In Origami, an interface is a set of meta-data, contracts, and an unique identifier. For example, an interface for a Set may be specified as follows:

```
interface Data.Set

uses Data.Boolean exposing Boolean

type Set<T>

function empty<T>() :: Set<T>
function add(set :: Set<T>, value :: T) :: Set<T>
function has(set :: Set<T>, value :: T) :: Boolean
```

This code describes the interface uniquely identified by `Data.Set`, which depends on the interface `Data.Boolean` providing a type with the name `Boolean`.

Implementations of `Data.Set` are required to provide an abstract and generic `Set<T>` type, and three operations on such type: `empty`, `add` and `has`. Here's one possible implementation:

```
module Data.Set

uses Data.Boolean exposing Boolean

union Set<T> {
  Empty
  Has(value :: T, rest :: Set<T>)
}

function empty<T>() :: Set<T> =
  Set.Empty;

function add(set :: Set<T>, value :: T) :: Set<T> =
  Set.Has(value, set);

function has(set :: Set<T>, value :: T) :: Boolean =
  match set {
    case Set.Empty: false;
    case Set.Has(v, rest):
      if v == value then true
      else has(rest, value);
  }
```

Here we have a module that implements the interface `Data.Set` (the module itself has no name, and it's not possible to refer to it directly), and it also depends on the interface `Data.Boolean`. It implements `Set<T>` as an union type.

This second module could also exist in the same program, at the same time:

```
module Data.Set

type Set<T> = (T) -> Boolean

function empty<T>() :: Set<T> =
  (_) => false;

function add(set :: Set<T>, value :: T) :: Set<T> =
  (v) => if v == value then true
         else set(value);

function has(set :: Set<T>, value :: T) :: Boolean =
  set(value);
```

Which implements the interface `Data.Set` with `Set<T>` being a simple closure.

Now, we have two modules in the system that implement the same interface, in _behaviourally-equivalent_ ways, but not _value-equivalent_ ways (you cannot pass a set from the first module into a function of the second module). But there's still a question of how one goes about _using_ one of these modules, as referring to any specific module is not possible in this system.

Using a module is a _linking_ problem. Modules describe only dependencies in terms of interfaces, and it's the job of the linker to provide them with actual modules that fulfill their expectations.

The simplest way in which the linker works is by looking at the interface identifier, and looking for a module that implements it, then providing that module. This works well in the scenario where there's only one implementation of a particular module, but that's not the case here: we have two modules implementing the same interface.

For ambiguous cases, the linker needs additional _constraints_ to disambiguate the linking. Constraints may be provided either directly in the source form (by specifying any meta-data attached to the module), or externally in the application's description--a special file that configures the application.

So if we have:

```
module App

uses Data.Set as Set

function main(_) =
  Set.empty()
  |> Set.add(_, 1)
  |> Set.add(_, 2);
```

We would need an application description like this:

```
package {
  entry: App;

  link {
    Data.Set => "path/to/data.set/using/unions"
  }
}
```

The `link` description disambiguates linking by specifying the actual implementation to use for an interface. These specifications may also use other constraints (for example, the author of a particular module), and they may be divided into "search spaces", which can then be applied to groups of modules.

Search spaces allow trust to be managed in an easier way. For example, one can define a "low trust" search space that does not have access to most effectful modules, such that an attacker wouldn't be able to send information over the network, or access the file system, even if they were able to take over some dependency you use. And a "high trust" search space for packages owned by the user, that has access to everything the program has.

## Interfaces

An interface describes the _minimal_ set of behaviours that must be implemented, by describing a set of contracts. Interfaces may be seen as the following:

```
interface ::= Id * Meta ... * Signature ...

Signature ::=
  | Define: name * type
  | Function: name * arity * type
  | Union: name * [name : arity] ...
  | Record: name * key ...
  | Type: name * type
```

So an interface is a triple of an unique identifier, a set of meta-data (for example, author, trust-level, stability, etc), and a set of signatures. For signatures we only care about the shape of the definition for equivalence. We don't try to prove that two modules have equivalent _types_, as those are defined in terms of higher-order contracts.

As an example, the boolean interface could be defined as such:

```
@Experimental
interface Data.Boolean

uses Origami.Annotation.Stability exposing Experimental

/**
 * The boolean data type.
 */
union Boolean {
  True
  False
}

/**
 * Logical conjunction.
 */
function (l :: Boolean) and (r :: Boolean) :: Boolean

/**
 * Logical disjunction.
 */
function (l :: Boolean) or (r :: Boolean) :: Boolean

/**
 * Logical negation.
 */
function not (v :: Boolean) :: Boolean
```

This interface has the unique identifier `Data.Boolean`, a metadata of `Origami.Annotation.Stability.Experimental`, and four signatures: `Union(Boolean, [True, False])`, `_ and _`, `_ or _`, and `not _`.

## Modules

A module implements an interface through its unique identifier, and must define at least all of the signatures described in the interface. A module _may_ define more signatures than what's expected by the interface. Where this is the case, those signatures are accessible within the module, but not outside of it--as other modules may only depend on the interface.

A module may be described as follows:

```
Module ::= Id * Meta ... * Definition ...

Definition ::=
  | Define: Name * type * Expression
  | Function: Name * Name ... * Type * Statement ...
  | Union: Name * [Name : Field ...] ...
  | Record: Name * Field ...
  | Type: Name * Type

Field ::= Name * Type * Expression?
```

Which is pretty close to what interfaces support, but include an implementation for all of the definitions. Unions and Records may define default values for fields in a module, which is not supported in interfaces--interfaces may not contain expressions.

All of the definitions in a module are late bound. They are evaluated only when (and if) they're used. This allows the system to fully analyse a module to figure out if it implements an interface correctly or not, and link the appropriate definitions, without executing any code, thus avoiding some security problems from module loading, and supporting mutually-recursive modules.

An implementation of the previously-specified `Data.Boolean` interface would look as follows:

```
@Author("Quil")
module Data.Boolean

uses Origami.Annotation.Author exposing Author

union Boolean { True; False }

type True = Boolean.True
type False = Boolean.False

function (_ :: True) and (_ :: True) = True
function _ and _ = False

function (_ :: True) or _ = True
function _ or (_ :: True) = True
function _ or _ = False

function not (_ :: True) = False
function not (_ :: False) = True
```

The implementation provides a definition for every signature in the interface, and defines two additional types. As these types don't appear in the interface, they're local to the module, and can't be accessed from the outside.

The implementation also attaches an Author annotation to the module. The module inherits the annotations from the interface, and so contains both the Stability annotation and the Author one. These annotations may be used as constraints for the linker.

## Annotations

Annotations are a special form of data structure that are associated directly with an interface. For annotations, we don't actually _link_ a module implementation, but rather use the structure defined in the interface. In fact, modules can't even define annotations, as that could lead to conflicts if two modules defined incompatible annotations with the same name.

Other than that, an annotations is not very different from a `record` definition. The major difference is that annotations don't have any real runtime representation, being a compile-only construct.

## Dependencies

Interfaces may depend on other interfaces. Because intefaces are identified by unique names, resolving these dependencies is simple enough: we just look at the dictionary of names to interfaces to resolve them:

```
interface(id) :: Interface
```

Dependencies in modules are trickier: they define an interface, together with some constraints, and something must translate this into an actual module implementation satisfying the expectations of the programmer of that module. We do this with a _linker_, an operation that takes in a module, an interface identifier, and some constraints, and resolves the dependencies in the module to actual modules that have been registered.

### Dependency constraints

The linker translates dependencies into a _constraint_. It then uses this constraint to find suitable modules to link.

For example, given the dependency:

```
uses Data.Boolean exposing Boolean, _ and _
```

The linker considers the constraint:

```
Id = Data.Boolean /\ Has(Define(Boolean)) /\ Has(Function(and, 2))
```

And given the dependency:

```
uses Data.Boolean as b
  if Author("Quil")
```

It considers the constraint:

```
Id = Data.Boolean /\ Meta(Origami.Annotation.Author.Author("Quil"))
```

The following is the constraint language used by the linker:

```
n in Number
s in String
b in Boolean
l in Label
v in Variable

Constraint c ::=
  | Id = <id>           -- interface identifier
  | Has(signature)      -- exposes signature
  | Meta(meta(r..))     -- contains or inherits meta with values
                           satisfying the given relation
  | c /\ c              -- conjunction
  | c \/ c              -- disjunction
  | ~c                  -- negation

Relation r ::=
  | n | s | b               -- the value is the primitive
  | v                       -- any value captured as v
  | r.l                     -- projection
  | r /\ r                  -- conjunction
  | r \/ r                  -- disjunction
  | ~r                      -- negation
  | r = p | r > p | r < p   -- relations
```

This language is powerful enough to include most common categories, without causing problems such as non-termination. Evaluation is done by unification, as in relational logic.

For example, a constraint may be given for the version of a module:

```
uses Origami.Annotations.Version exposing Version
uses Data.List as L
  if Version(v.major = 2 & v.feature > 3)
```

Would be the equivalent of "only link modules whose version is '>2.3 and <3.0'".

### Search spaces

(TODO:)

### Ambiguity

(TODO:)

## Security

(TODO:)

## Packaging

(TODO:)

## Practical affordances

### Generating interfaces from modules

As specified, the module system would require that programmers maintained both an interface description _and_ an implementation. In separate files. This is not really reasonable for most modules, where there's no expectation that the module will be implemented more than once. Most application modules are really application-specific, and would not benefit from this.

To avoid this problem, we allow interfaces to be generated from module definitions **if** there's only one module for such interface in the program. This way people only need to define an interface file for the types of modules for which they either want to share with other programmers, or for which they expect multiple implementations to exist.

### Caching & pinning linked modules

In order to execute an Origami application, the linker must resolve all of the constraints in dependencies to provide actual module implementations to every module. The cost of resolving these constraints grows with the number of modules and dependencies. On top of that, if the module files change, a different module may be linked _even if_ the constraints themselves haven't changed. If a linked module changes after programmers have executed the program a few times, the behaviour could be confusing.

To avoid this we allow linked modules to be pinned and cached. Which means that after we resolve the constraint once for a module, we will reuse the module previously linked until the programmer explicitly asks the linker to update the linked module--in which case we'll resolve the constraints again for that module.

Note that we _still_ have to check that the linked modules fulfill all of the requirements specified in the dependency. The only cost we avoid here is looking at the other modules in the search space.

## TODO: Open questions

- Types and generics (the type system is still not defined);
- The application description format is not defined;
- Threat modelling;
- What happens if people plug values from different implementations into some function?