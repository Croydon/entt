# Crash Course: entity-component system

<!--
@cond TURN_OFF_DOXYGEN
-->
# Table of Contents

* [Introduction](#introduction)
* [Design choices](#design-choices)
  * [A bitset-free entity-component system](#a-bitset-free-entity-component-system)
  * [Pay per use](#pay-per-use)
* [Vademecum](#vademecum)
* [The Registry, the Entity and the Component](#the-registry-the-entity-and-the-component)
  * [Observe changes](#observe-changes)
  * [Runtime components](#runtime-components)
    * [A journey through a plugin](#a-journey-through-a-plugin)
  * [Sorting: is it possible?](#sorting-is-it-possible)
  * [Snapshot: complete vs continuous](#snapshot-complete-vs-continuous)
    * [Snapshot loader](#snapshot-loader)
    * [Continuous loader](#continuous-loader)
    * [Archives](#archives)
    * [One example to rule them all](#one-example-to-rule-them-all)
  * [Prototype](#prototype)
  * [Helpers](#helpers)
    * [Dependency function](#dependency-function)
    * [Labels](#labels)
  * [Null entity](#null-entity)
* [Views: pay for what you use](#views-pay-for-what-you-use)
  * [Standard View](#standard-view)
    * [Single component standard view](#single-component-standard-view)
    * [Multi component standard view](#multi-component-standard-view)
  * [Persistent View](#persistent-view)
  * [Raw View](#raw-view)
  * [Runtime View](#runtime-view)
  * [Types: const, non-const and all in between](#types-const-non-const-and-all-in-between)
  * [Give me everything](#give-me-everything)
* [Iterations: what is allowed and what is not](#iterations-what-is-allowed-and-what-is-not)
* [Multithreading](#multithreading)
  * [Views and Iterators](#views-and-iterators)
<!--
@endcond TURN_OFF_DOXYGEN
-->

# Introduction

`EnTT` is a header-only, tiny and easy to use entity-component system (and much
more) written in modern C++.<br/>
The entity-component-system (also known as _ECS_) is an architectural pattern
used mostly in game development.

# Design choices

## A bitset-free entity-component system

`EnTT` is a _bitset-free_ entity-component system that doesn't require users to
specify the component set at compile-time.<br/>
This is why users can instantiate the core class simply like:

```cpp
entt::registry registry;
```

In place of its more annoying and error-prone counterpart:

```cpp
entt::registry<comp_0, comp_1, ..., comp_n> registry;
```

## Pay per use

`EnTT` is entirely designed around the principle that users have to pay only for
what they want.

When it comes to using an entity-component system, the tradeoff is usually
between performance and memory usage. The faster it is, the more memory it uses.
However, slightly worse performance along non-critical paths are the right price
to pay to reduce memory usage and I've always wondered why this kind of tools do
not leave me the choice.<br/>
`EnTT` follows a completely different approach. It squeezes the best from the
basic data structures and gives users the possibility to pay more for higher
performance where needed.<br/>
The disadvantage of this approach is that users need to know the systems they
are working on and the tools they are using. Otherwise, the risk to ruin the
performance along critical paths is high.

So far, this choice has proven to be a good one and I really hope it can be for
many others besides me.

# Vademecum

The registry to store, the views to iterate. That's all.

An entity (the _E_ of an _ECS_) is an opaque identifier that users should just
use as-is and store around if needed. Do not try to inspect an entity
identifier, its format can change in future and a registry offers all the
functionalities to query them out-of-the-box. The underlying type of an entity
(either `std::uint16_t`, `std::uint32_t` or `std::uint64_t`) can be specified
when defining a registry (actually `registry` is nothing more than an _alias_
for `registry<std::uint32_t>`).<br/>
Components (the _C_ of an _ECS_) should be plain old data structures or more
complex and movable data structures with a proper constructor. Actually, the
sole requirement of a component type is that it must be both move constructible
and move assignable. They are list initialized by using the parameters provided
to construct the component itself. No need to register components or their types
neither with the registry nor with the entity-component system at all.<br/>
Systems (the _S_ of an _ECS_) are just plain functions, functors, lambdas or
whatever users want. They can accept a `registry` or a view of any type and use
them the way they prefer. No need to register systems or their types neither
with the registry nor with the entity-component system at all.

The following sections will explain in short how to use the entity-component
system, the core part of the whole library.<br/>
In fact, the project is composed of many other classes in addition to those
describe below. For more details, please refer to the inline documentation.

# The Registry, the Entity and the Component

A registry can store and manage entities, as well as create views to iterate the
underlying data structures.<br/>
The class template `registry` lets users decide what's the preferred type to
represent an entity. Because `std::uint32_t` is large enough for almost all the
cases, `registry` is also an _alias_ for `registry<std::uint32_t>`.

Entities are represented by _entity identifiers_. An entity identifier is an
opaque type that users should not inspect or modify in any way. It carries
information about the entity itself and its version.

A registry can be used both to construct and destroy entities:

```cpp
// constructs a naked entity with no components and returns its identifier
auto entity = registry.create();

// destroys an entity and all its components
registry.destroy(entity);
```

There exists another overload of the `create` member function that accepts two
iterators, that is a range to assign. It can be used to create multiple entities
at once.<br/>
Entities can also be destroyed _by type_, that is by specifying the types of the
components that identify them:

```cpp
// destroys the entities that own the given components, if any
registry.destroy<a_component, another_component>();
```

When an entity is destroyed, the registry can freely reuse it internally with a
slightly different identifier. In particular, the version of an entity is
increased each and every time it's discarded.<br/>
In case entity identifiers are stored around, the registry offers all the
functionalities required to test them and get out of the them all the
information they carry:

```cpp
// returns true if the entity is still valid, false otherwise
bool b = registry.valid(entity);

// gets the version contained in the entity identifier
auto version = registry.version(entity);

// gets the actual version for the given entity
auto curr = registry.current(entity);
```

Components can be assigned to or removed from entities at any time with a few
calls to member functions of the registry. As for the entities, the registry
offers also a set of functionalities users can use to work with the components.

The `assign` member function template creates, initializes and assigns to an
entity the given component. It accepts a variable number of arguments to
construct the component itself if present:

```cpp
registry.assign<position>(entity, 0., 0.);

// ...

auto &velocity = registry.assign<velocity>(entity);
vel.dx = 0.;
vel.dy = 0.;
```

If an entity already has the given component, the `replace` member function
template can be used to replace it:

```cpp
registry.replace<position>(entity, 0., 0.);

// ...

auto &velocity = registry.replace<velocity>(entity);
vel.dx = 0.;
vel.dy = 0.;
```

In case users want to assign a component to an entity, but it's unknown whether
the entity already has it or not, `assign_or_replace` does the work in a single
call (there is a performance penalty to pay for this mainly due to the fact that
it has to check if the entity already has the given component or not):

```cpp
registry.assign_or_replace<position>(entity, 0., 0.);

// ...

auto &velocity = registry.assign_or_replace<velocity>(entity);
vel.dx = 0.;
vel.dy = 0.;
```

Note that `assign_or_replace` is a slightly faster alternative for the following
`if/else` statement and nothing more:

```cpp
if(registry.has<comp>(entity)) {
    registry.replace<comp>(entity, arg1, argN);
} else {
    registry.assign<comp>(entity, arg1, argN);
}
```

As already shown, if in doubt about whether or not an entity has one or more
components, the `has` member function template may be useful:

```cpp
bool b = registry.has<position, velocity>(entity);
```

On the other side, if the goal is to delete a single component, the `remove`
member function template is the way to go when it's certain that the entity owns
a copy of the component:

```cpp
registry.remove<position>(entity);
```

Otherwise consider to use the `reset` member function. It behaves similarly to
`remove` but with a strictly defined behavior (and a performance penalty is the
price to pay for this). In particular it removes the component if and only if it
exists, otherwise it returns safely to the caller:

```cpp
registry.reset<position>(entity);
```

There exist also two other _versions_ of the `reset` member function:

* If no entity is passed to it, `reset` will remove the given component from
  each entity that has it:

  ```cpp
  registry.reset<position>();
  ```

* If neither the entity nor the component are specified, all the entities still
  in use and their components are destroyed:

  ```cpp
  registry.reset();
  ```

Finally, references to components can be retrieved simply by doing this:

```cpp
const auto &cregistry = registry;

// const and non-const reference
const auto &crenderable = cregistry.get<renderable>(entity);
auto &renderable = registry.get<renderable>(entity);

// const and non-const references
const auto &[cpos, cvel] = cregistry.get<position, velocity>(entity);
auto &[pos, vel] = registry.get<position, velocity>(entity);
```

The `get` member function template gives direct access to the component of an
entity stored in the underlying data structures of the registry. There exists
also an alternative member function named `try_get` that returns a pointer to
the component owned by an entity if any, a null pointer otherwise.

## Observe changes

Because of how the registry works internally, it stores a couple of signal
handlers for each pool in order to notify some of its data structures on the
construction and destruction of components.<br/>
These signal handlers are also exposed and made available to users. This is the
basic brick to build fancy things like dependencies and reactive systems.

To get a sink to be used to connect and disconnect listeners so as to be
notified on the creation of a component, use the `construction` member function:

```cpp
// connects a free function
registry.construction<position>().connect<&my_free_function>();

// connects a member function
registry.construction<position>().connect<&my_class::member>(&instance);

// disconnects a free function
registry.construction<position>().disconnect<&my_free_function>();

// disconnects a member function
registry.construction<position>().disconnect<&my_class::member>(&instance);
```

To be notified when components are destroyed, use the `destruction` member
function instead.

The function type of a listener is the same in both cases and should be
equivalent to:

```cpp
void(registry<Entity> &, Entity);
```

In other terms, a listener is provided with the registry that triggered the
notification and the entity affected by the change. Note also that:

* Listeners are invoked **after** components have been assigned to entities.
* Listeners are invoked **before** components have been removed from entities.
* The order of invocation of the listeners isn't guaranteed in any case.

There are also some limitations on what a listener can and cannot do. In
particular:

* Connecting and disconnecting other functions from within the body of a
  listener should be avoided. It can lead to undefined behavior in some cases.
* Assigning and removing components from within the body of a listener that
  observes the destruction of instances of a given type should be avoided. It
  can lead to undefined behavior in some cases. This type of listeners is
  intended to provide users with an easy way to perform cleanup and nothing
  more.

To a certain extent, these limitations do not apply. However, it is risky to try
to force them and users should respect the limitations unless they know exactly
what they are doing. Subtle bugs are the price to pay in case of errors
otherwise.

In general, events and therefore listeners must not be used as replacements for
systems. They should not contain much logic and interactions with a registry
should be kept to a minimum, if possible. Note also that the greater the number
of listeners, the greater the performance hit when components are created or
destroyed.

## Runtime components

Defining components at runtime is useful to support plugin systems and mods in
general. However, it seems impossible with a tool designed around a bunch of
templates. Indeed it's not that difficult.<br/>
Of course, some features cannot be easily exported into a runtime
environment. As an example, sorting a group of components defined at runtime
isn't for free if compared to most of the other operations. However, the basic
functionalities of an entity-component system such as `EnTT` fit the problem
perfectly and can also be used to manage runtime components if required.<br/>
All that is necessary to do it is to know the identifiers of the components. An
identifier is nothing more than a number or similar that can be used at runtime
to work with the type system.

In `EnTT`, identifiers are easily accessible:

```cpp
entt::registry registry;

// component identifier
auto type = registry.type<position>();
```

Once the identifiers are made available, almost everything becomes pretty
simple.

### A journey through a plugin

`EnTT` comes with an example (actually a test) that shows how to integrate
compile-time and runtime components in a stack based JavaScript environment. It
uses [`Duktape`](https://github.com/svaarala/duktape) under the hood, mainly
because I wanted to learn how it works at the time I was writing the code.

The code is not production-ready and overall performance can be highly improved.
However, I sacrificed optimizations in favor of a more readable piece of code. I
hope I succeeded.<br/>
Note also that this isn't neither the only nor (probably) the best way to do it.
In fact, the right way depends on the scripting language and the problem one is
facing in general.<br/>
That being said, feel free to use it at your own risk.

The basic idea is that of creating a compile-time component aimed to map all the
runtime components assigned to an entity.<br/>
Identifiers come in use to address the right function from a map when invoked
from the runtime environment and to filter entities when iterating.<br/>
With a bit of gymnastic, one can narrow views and improve the performance to
some extent but it was not the goal of the example.

## Sorting: is it possible?

It goes without saying that sorting entities and components is possible with
`EnTT`.<br/>
In fact, there are two functions that respond to slightly different needs:

* Components can be sorted directly:

  ```cpp
  registry.sort<renderable>([](const auto &lhs, const auto &rhs) {
      return lhs.z < rhs.z;

  });
  ```

  There exists also the possibility to use a custom sort function object, as
  long as it adheres to the requirements described in the inline
  documentation.<br/>
  This is possible mainly because users can get much more with a custom sort
  function object if the pattern of usage is known. As an example, in case of an
  almost sorted pool, quick sort could be much, much slower than insertion sort.

* Components can be sorted according to the order imposed by another component:

  ```cpp
  registry.sort<movement, physics>();
  ```

  In this case, instances of `movement` are arranged in memory so that cache
  misses are minimized when the two components are iterated together.

## Snapshot: complete vs continuous

The `registry` class offers basic support to serialization.<br/>
It doesn't convert components to bytes directly, there wasn't the need of
another tool for serialization out there. Instead, it accepts an opaque object
with a suitable interface (namely an _archive_) to serialize its internal data
structures and restore them later. The way types and instances are converted to
a bunch of bytes is completely in charge to the archive and thus to final users.

The goal of the serialization part is to allow users to make both a dump of the
entire registry or a narrower snapshot, that is to select only the components in
which they are interested.<br/>
Intuitively, the use cases are different. As an example, the first approach is
suitable for local save/restore functionalities while the latter is suitable for
creating client-server applications and for transferring somehow parts of the
representation side to side.

To take a snapshot of the registry, use the `snapshot` member function. It
returns a temporary object properly initialized to _save_ the whole registry or
parts of it.

Example of use:

```cpp
output_archive output;

registry.snapshot()
    .entities(output)
    .destroyed(output)
    .component<a_component, another_component>(output);
```

It isn't necessary to invoke all these functions each and every time. What
functions to use in which case mostly depends on the goal and there is not a
golden rule to do that.

The `entities` member function asks the registry to serialize all the entities
that are still in use along with their versions. On the other side, the
`destroyed` member function tells to the registry to serialize the entities that
have been destroyed and are no longer in use.<br/>
These two functions can be used to save and restore the whole set of entities
with the versions they had during serialization.

The `component` member function is a function template the aim of which is to
store aside components. The presence of a template parameter list is a
consequence of a couple of design choices from the past and in the present:

* First of all, there is no reason to force a user to serialize all the
  components at once and most of the times it isn't desiderable. As an example,
  in case the stuff for the HUD in a game is put into the registry for some
  reasons, its components can be freely discarded during a serialization step
  because probably the software already knows how to reconstruct the HUD
  correctly from scratch.

* Furthermore, the registry makes heavy use of _type-erasure_ techniques
  internally and doesn't know at any time what types of components it contains.
  Therefore being explicit at the call point is mandatory.

There exists also another version of the `component` member function that
accepts a range of entities to serialize. This version is a bit slower than the
other one, mainly because it iterates the range of entities more than once for
internal purposes. However, it can be used to filter out those entities that
shouldn't be serialized for some reasons.<br/>
As an example:

```cpp
const auto view = registry.view<serialize>();
output_archive output;

registry.snapshot().component<a_component, another_component>(output, view.cbegin(), view.cend());
```

Note that `component` stores items along with entities. It means that it works
properly without a call to the `entities` member function.

Once a snapshot is created, there exist mainly two _ways_ to load it: as a whole
and in a kind of _continuous mode_.<br/>
The following sections describe both loaders and archives in details.

### Snapshot loader

A snapshot loader requires that the destination registry be empty and loads all
the data at once while keeping intact the identifiers that the entities
originally had.<br/>
To do that, the registry offers a member function named `loader` that returns a
temporary object properly initialized to _restore_ a snapshot.

Example of use:

```cpp
input_archive input;

registry.loader()
    .entities(input)
    .destroyed(input)
    .component<a_component, another_component>(input)
    .orphans();
```

It isn't necessary to invoke all these functions each and every time. What
functions to use in which case mostly depends on the goal and there is not a
golden rule to do that. For obvious reasons, what is important is that the data
are restored in exactly the same order in which they were serialized.

The `entities` and `destroyed` member functions restore the sets of entities and
the versions that the entities originally had at the source.

The `component` member function restores all and only the components specified
and assigns them to the right entities. Note that the template parameter list
must be exactly the same used during the serialization.

The `orphans` member function literally destroys those entities that have no
components attached. It's usually useless if the snapshot is a full dump of the
source. However, in case all the entities are serialized but only few components
are saved, it could happen that some of the entities have no components once
restored. The best users can do to deal with them is to destroy those entities
and thus update their versions.

### Continuous loader

A continuous loader is designed to load data from a source registry to a
(possibly) non-empty destination. The loader can accommodate in a registry more
than one snapshot in a sort of _continuous loading_ that updates the
destination one step at a time.<br/>
Identifiers that entities originally had are not transferred to the target.
Instead, the loader maps remote identifiers to local ones while restoring a
snapshot. Because of that, this kind of loader offers a way to update
automatically identifiers that are part of components (as an example, as data
members or gathered in a container).<br/>
Another difference with the snapshot loader is that the continuous loader does
not need to work with the private data structures of a registry. Furthermore, it
has an internal state that must persist over time. Therefore, there is no reason
to create it by means of a registry, or to limit its lifetime to that of a
temporary object.

Example of use:

```cpp
entt::continuous_loader<entity_type> loader{registry};
input_archive input;

loader.entities(input)
    .destroyed(input)
    .component<a_component, another_component, dirty_component>(input, &dirty_component::parent, &dirty_component::child)
    .orphans()
    .shrink();
```

It isn't necessary to invoke all these functions each and every time. What
functions to use in which case mostly depends on the goal and there is not a
golden rule to do that. For obvious reasons, what is important is that the data
are restored in exactly the same order in which they were serialized.

The `entities` and `destroyed` member functions restore groups of entities and
map each entity to a local counterpart when required. In other terms, for each
remote entity identifier not yet registered by the loader, the latter creates a
local identifier so that it can keep the local entity in sync with the remote
one.

The `component` member function restores all and only the components specified
and assigns them to the right entities.<br/>
In case the component contains entities itself (either as data members of type
`entity_type` or as containers of entities), the loader can update them
automatically. To do that, it's enough to specify the data members to update as
shown in the example.

The `orphans` member function literally destroys those entities that have no
components after a restore. It has exactly the same purpose described in the
previous section and works the same way.

Finally, `shrink` helps to purge local entities that no longer have a remote
conterpart. Users should invoke this member function after restoring each
snapshot, unless they know exactly what they are doing.

### Archives

Archives must publicly expose a predefined set of member functions. The API is
straightforward and consists only of a group of function call operators that
are invoked by the snapshot class and the loaders.

In particular:

* An output archive, the one used when creating a snapshot, must expose a
  function call operator with the following signature to store entities:

  ```cpp
  void operator()(Entity);
  ```

  Where `Entity` is the type of the entities used by the registry. Note that all
  the member functions of the snapshot class make also an initial call to this
  endpoint to save the _size_ of the set they are going to store.<br/>
  In addition, an archive must accept a pair of entity and component for each
  type to be serialized. Therefore, given a type `T`, the archive must contain a
  function call operator with the following signature:

  ```cpp
  void operator()(Entity, const T &);
  ```

  The output archive can freely decide how to serialize the data. The register
  is not affected at all by the decision.

* An input archive, the one used when restoring a snapshot, must expose a
  function call operator with the following signature to load entities:

  ```cpp
  void operator()(Entity &);
  ```

  Where `Entity` is the type of the entities used by the registry. Each time the
  function is invoked, the archive must read the next element from the
  underlying storage and copy it in the given variable. Note that all the member
  functions of a loader class make also an initial call to this endpoint to read
  the _size_ of the set they are going to load.<br/>
  In addition, the archive must accept a pair of entity and component for each
  type to be restored. Therefore, given a type `T`, the archive must contain a
  function call operator with the following signature:

  ```cpp
  void operator()(Entity &, T &);
  ```

  Every time such an operator is invoked, the archive must read the next
  elements from the underlying storage and copy them in the given variables.

### One example to rule them all

`EnTT` comes with some examples (actually some tests) that show how to integrate
a well known library for serialization as an archive. It uses
[`Cereal C++`](https://uscilab.github.io/cereal/) under the hood, mainly
because I wanted to learn how it works at the time I was writing the code.

The code is not production-ready and it isn't neither the only nor (probably)
the best way to do it. However, feel free to use it at your own risk.

The basic idea is to store everything in a group of queues in memory, then bring
everything back to the registry with different loaders.

## Prototype

A prototype defines a type of an application in terms of its parts. They can be
used to assign components to entities of a registry at once.<br/>
Roughly speaking, in most cases prototypes can be considered just as templates
to use to initialize entities according to _concepts_. In fact, users can create
how many prototypes they want, each one initialized differently from the others.

The following is an example of use of a prototype:

```cpp
entt::registry registry;
entt::prototype prototype{registry};

prototype.set<position>(100.f, 100.f);
prototype.set<velocity>(0.f, 0.f);

// ...

const auto entity = prototype();
```

To assign and remove components from a prototype, it offers two dedicated member
functions named `set` and `unset`. The `has` member function can be used to know
if a given prototype contains one or more components and the `get` member
function can be used to retrieve the components.

Creating an entity from a prototype is straightforward:

* To create a new entity from scratch and assign it a prototype, this is the way
  to go:
  ```cpp
  const auto entity = prototype();
  ```
  It is equivalent to the following invokation:
  ```cpp
  const auto entity = prototype.create();
  ```

* In case we want to initialize an already existing entity, we can provide the
  `operator()` directly with the entity identifier:
  ```cpp
  prototype(entity);
  ```
  It is equivalent to the following invokation:
  ```cpp
  prototype.assign(entity);
  ```
  Note that existing components aren't overwritten in this case. Only those
  components that the entity doesn't own yet are copied over. All the other
  components remain unchanged.

* Finally, to assign or replace all the components for an entity, thus
  overwriting existing ones:
  ```cpp
  prototype.assign_or_replace(entity);
  ```

In the examples above, the prototype uses its underlying registry to create
entities and components both for its purposes and when it's cloned. To use a
different repository to clone a prototype, all the member functions accept also
a reference to a valid registry as a first argument.

Prototypes are a very useful tool that can save a lot of typing sometimes.
Furthermore, the codebase may be easier to maintain, since updating a prototype
is much less error prone than jumping around in the codebase to update all the
snippets copied and pasted around to initialize entities and components.

## Helpers

The so called _helpers_ are small classes and functions mainly designed to offer
built-in support for the most basic functionalities.<br/>
The list of helpers will grow longer as time passes and new ideas come out.

### Dependency function

A _dependency function_ is a predefined listener, actually a function template
to use to automatically assign components to an entity when a type has a
dependency on some other types.<br/>
The following adds components `a_type` and `another_type` whenever `my_type` is
assigned to an entity:

```cpp
entt::connnect<a_type, another_type>(registry.construction<my_type>());
```

A component is assigned to an entity and thus default initialized only in case
the entity itself hasn't it yet. It means that already existent components won't
be overriden.<br/>
A dependency can easily be broken by means of the following function template:

```cpp
entt::disconnect<a_type, another_type>(registry.construction<my_type>());
```

### Labels

There's nothing magical about the way labels can be assigned to entities while
avoiding a performance hit at runtime. Nonetheless, the syntax can be annoying
and that's why a more user-friendly shortcut is provided to do it.<br/>
This shortcut is the alias template `entt::label`.

If used in combination with hashed strings, it helps to use labels where types
would be required otherwise. As an example:

```cpp
registry.assign<entt::label<"enemy"_hs>>(entity);
```

## Null entity

In `EnTT`, there exists a sort of _null entity_ made available to users that is
accessible via the `entt::null` variable.<br/>
The library guarantees that the following expression always returns false:

```cpp
registry.valid(entt::null);
```

In other terms, a registry will reject the null entity in all cases because it
isn't considered valid. It means that the null entity cannot own components for
obvious reasons.<br/>
The type of the null entity is internal and should not be used for any purpose
other than defining the null entity itself. However, there exist implicit
conversions from the null entity to identifiers of any allowed type:

```cpp
typename entt::registry::entity_type null = entt::null;
```

Similarly, the null entity can be compared to any other identifier:

```cpp
const auto entity = registry.create();
const bool null = (entity == entt::null);
```

# Views: pay for what you use

First of all, it is worth answering an obvious question: why views?<br/>
Roughly speaking, they are a good tool to enforce single responsibility. A
system that has access to a registry can create and destroy entities, as well as
assign and remove components. On the other side, a system that has access to a
view can only iterate entities and their components, then read or update the
data members of the latter.<br/>
It is a subtle difference that can help designing a better software sometimes.

There are mainly four kinds of views: standard (also known as `view`),
persistent (also known as `persistent_view`), raw (also known as `raw_view`) and
runtime (also known as `runtime_view`).<br/>
All of them have pros and cons to take in consideration. In particular:

* Standard views:

  Pros:

  * They work out-of-the-box and don't require any dedicated data structure.
  * Creating and destroying them isn't expensive at all because they don't have
    any type of initialization.
  * They are the best tool for iterating a single component.
  * They are the best tool for iterating multiple components when one of them is
    assigned to a significantly low number of entities.
  * They don't affect any other operations of the registry.

  Cons:

  * Their performance tend to degenerate when the number of components to
    iterate grows up and the most of the entities have all of them.

* Persistent views:

  Pros:

  * Once prepared, creating and destroying them isn't expensive at all because
    they don't have any type of initialization.
  * They are the best tool for iterating multiple components when most entities
    have them all.

  Cons:

  * They have dedicated data structures and thus affect the memory usage to a
    minimal extent.
  * If not previously initialized, the first time they are used they go through
    an initialization step that is slightly more expensive.
  * They affect to a minimum the creation and destruction of entities and
    components, as well as the sort functionalities. In other terms: the more
    persistent views there will be, the less performing will be creating and
    destroying entities and components or sorting a pool.

* Raw views:

  Pros:

  * They work out-of-the-box and don't require any dedicated data structure.
  * Creating and destroying them isn't expensive at all because they don't have
    any type of initialization.
  * They are the best tool for iterating components when it is not necessary to
    know which entities they belong to.
  * They don't affect any other operations of the registry.

  Cons:

  * They can be used to iterate only one type of component at a time.
  * They don't return the entity to which a component belongs to the caller.

* Runtime views:

  Pros:

  * Their lists of components are defined at runtime and not at compile-time.
  * Creating and destroying them isn't expensive at all because they don't have
    any type of initialization.
  * They are the best tool for things like plugin systems and mods in general.
  * They don't affect any other operations of the registry.

  Cons:

  * Their performances are definitely lower than those of all the other views,
    although they are still usable and sufficient for most of the purposes.

To sum up and as a rule of thumb:

* Use a raw view to iterate components only (no entities) for a given type.
* Use a standard view to iterate entities and components for a single type.
* Use a standard view to iterate entities and components for multiple types when
  a significantly low number of entities have one of the components, persistent
  views won't add much in this case.
* Use a standard view in all those cases where a persistent view would give a
  boost to performance but the iteration isn't performed frequently or isn't on
  a critical path.
* Use a persistent view when you want to iterate multiple components and each
  component is assigned to a great number of entities but the intersection
  between the sets of entities is small.
* Use a persistent view in all the cases where a standard view wouldn't fit well
  otherwise.
* Finally, in case you don't know at compile-time what are the components to
  use, choose a runtime view and set them during execution.

To easily iterate entities and components, all the views offer the common
`begin` and `end` member functions that allow users to use a view in a typical
range-for loop. Almost all the views offer also a *more functional* `each`
member function that accepts a callback for convenience.<br/>
Continue reading for more details or refer to the inline documentation.

## Standard View

A standard view behaves differently if it's constructed for a single component
or if it has been requested to iterate multiple components. Even the API is
different in the two cases.<br/>
All that they share is the way they are created by means of a registry:

```cpp
// single component standard view
auto single = registry.view<position>();

// multi component standard view
auto multi = registry.view<position, velocity>();
```

For all that remains, it's worth discussing them separately.<br/>

### Single component standard view

Single component standard views are specialized in order to give a boost in
terms of performance in all the situation. This kind of views can access the
underlying data structures directly and avoid superfluous checks.<br/>
They offer a bunch of functionalities to get the number of entities they are
going to return and a raw access to the entity list as well as to the component
list. It's also possible to ask a view if it contains a given entity.<br/>
Refer to the inline documentation for all the details.

There is no need to store views around for they are extremely cheap to
construct, even though they can be copied without problems and reused freely. In
fact, they return newly created and correctly initialized iterators whenever
`begin` or `end` are invoked.<br/>
To iterate a single component standard view, either use it in a range-for loop:

```cpp
auto view = registry.view<renderable>();

for(auto entity: view) {
    renderable &renderable = view.get(entity);

    // ...
}
```

Or rely on the `each` member function to iterate entities and get all their
components at once:

```cpp
registry.view<renderable>().each([](auto entity, auto &renderable) {
    // ...
});
```

The `each` member function is highly optimized. Unless users want to iterate
only entities, using `each` should be the preferred approach.

**Note**: prefer the `get` member function of a view instead of the `get` member
function template of a registry during iterations, if possible. However, keep in
mind that it works only with the components of the view itself.

### Multi component standard view

Multi component standard views iterate entities that have at least all the given
components in their bags. During construction, these views look at the number of
entities available for each component and pick up a reference to the smallest
set of candidates in order to speed up iterations.<br/>
They offer fewer functionalities than their companion views for single
component. In particular, a multi component standard view exposes utility
functions to get the estimated number of entities it is going to return and to
know whether it's empty or not. It's also possible to ask a view if it contains
a given entity.<br/>
Refer to the inline documentation for all the details.

There is no need to store views around for they are extremely cheap to
construct, even though they can be copied without problems and reused freely. In
fact, they return newly created and correctly initialized iterators whenever
`begin` or `end` are invoked.<br/>
To iterate a multi component standard view, either use it in a range-for loop:

```cpp
auto view = registry.view<position, velocity>();

for(auto entity: view) {
    // a component at a time ...
    auto &position = view.get<position>(entity);
    auto &velocity = view.get<velocity>(entity);

    // ... or multiple components at once
    auto &[pos, vel] = view.get<position, velocity>(entity);

    // ...
}
```

Or rely on the `each` member function to iterate entities and get all their
components at once:

```cpp
registry.view<position, velocity>().each([](auto entity, auto &pos, auto &vel) {
    // ...
});
```

The `each` member function is highly optimized. Unless users want to iterate
only entities or get only some of the components, using `each` should be the
preferred approach.

**Note**: prefer the `get` member function of a view instead of the `get` member
function template of a registry during iterations, if possible. However, keep in
mind that it works only with the components of the view itself.

## Persistent View

A persistent view returns all the entities and only the entities that have at
least the given components. Moreover, it's guaranteed that the entity list is
tightly packed in memory for fast iterations.<br/>
In general, persistent views don't stay true to the order of any set of
components unless users explicitly sort them.

Persistent views can be used only to iterate multiple components at once:

```cpp
auto view = registry.persistent_view<position, velocity>();
```

There is no need to store views around for they are extremely cheap to
construct, even though they can be copied without problems and reused freely. In
fact, they return newly created and correctly initialized iterators whenever
`begin` or `end` are invoked.<br/>
That being said, persistent views perform an initialization step the very first
time they are constructed and this could be quite costly. To avoid it, consider
creating them when no components have been assigned yet. If the registry is
empty, preparation is extremely fast.

A persistent view offers a bunch of functionalities to get the number of
entities it's going to return, a raw access to the entity list and the
possibility to sort the underlying data structures according to the order of one
of the components for which it has been constructed. It's also possible to ask a
view if it contains a given entity.<br/>
Refer to the inline documentation for all the details.

To iterate a persistent view, either use it in a range-for loop:

```cpp
auto view = registry.persistent_view<position, velocity>();

for(auto entity: view) {
    // a component at a time ...
    auto &position = view.get<position>(entity);
    auto &velocity = view.get<velocity>(entity);

    // ... or multiple components at once
    auto &[pos, vel] = view.get<position, velocity>(entity);

    // ...
}
```

Or rely on the `each` member function to iterate entities and get all their
components at once:

```cpp
registry.persistent_view<position, velocity>().each([](auto entity, auto &pos, auto &vel) {
    // ...
});
```

The `each` member function is highly optimized. Unless users want to iterate
only entities, using `each` should be the preferred approach.

**Note**: prefer the `get` member function of a view instead of the `get` member
function template of a registry during iterations, if possible. However, keep in
mind that it works only with the components of the view itself.

## Raw View

Raw views return all the components of a given type. This kind of views can
access components directly and avoid extra indirections like when components are
accessed via an entity identifier.<br/>
They offer a bunch of functionalities to get the number of instances they are
going to return and a raw access to the entity list as well as to the component
list.<br/>
Refer to the inline documentation for all the details.

Raw views can be used only to iterate components for a single type:

```cpp
auto view = registry.raw_view<renderable>();
```

There is no need to store views around for they are extremely cheap to
construct, even though they can be copied without problems and reused freely. In
fact, they return newly created and correctly initialized iterators whenever
`begin` or `end` are invoked.<br/>
To iterate a raw view, use it in a range-for loop:

```cpp
auto view = registry.raw_view<renderable>();

for(auto &&component: raw) {
    // ...
}
```

Or rely on the `each` member function:

```cpp
registry.raw_view<renderable>().each([](auto &renderable) {
    // ...
});
```

Performance are exactly the same in both cases.

**Note**: raw views don't have a `get` member function for obvious reasons.

## Runtime View

Runtime views iterate entities that have at least all the given components in
their bags. During construction, these views look at the number of entities
available for each component and pick up a reference to the smallest
set of candidates in order to speed up iterations.<br/>
They offer more or less the same functionalities of a multi component standard
view. However, they don't expose a `get` member function and users should refer
to the registry that generated the view to access components. In particular, a
runtime view exposes utility functions to get the estimated number of entities
it is going to return and to know whether it's empty or not. It's also possible
to ask a view if it contains a given entity.<br/>
Refer to the inline documentation for all the details.

Runtime view are extremely cheap to construct and should not be stored around in
any case. They should be used immediately after creation and then they should be
thrown away. The reasons for this go far beyond the scope of this document.<br/>
To iterate a runtime view, either use it in a range-for loop:

```cpp
using component_type = typename decltype(registry)::component_type;
component_type types[] = { registry.type<position>(), registry.type<velocity>() };

auto view = registry.runtime_view(std::cbegin(types), std::cend(types));

for(auto entity: view) {
    // a component at a time ...
    auto &position = registry.get<position>(entity);
    auto &velocity = registry.get<velocity>(entity);

    // ... or multiple components at once
    auto &[pos, vel] = view.get<position, velocity>(entity);

    // ...
}
```

Or rely on the `each` member function to iterate entities:

```cpp
using component_type = typename decltype(registry)::component_type;
component_type types[] = { registry.type<position>(), registry.type<velocity>() };

auto view = registry.runtime_view(std::cbegin(types), std::cend(types)).each([](auto entity) {
    // ...
});
```

Performance are exactly the same in both cases.

**Note**: runtime views are meant for all those cases where users don't know at
compile-time what components to use to iterate entities. This is particularly
well suited to plugin systems and mods in general. Where possible, don't use
runtime views, as their performance are slightly inferior to those of the other
views.

# Types: const, non-const and all in between

The `registry` class offers two overloads for most of the member functions used
to construct views: a const one and a non-const one. The former accepts both
const and non-const types as template parameters, the latter accepts only const
types instead.<br/>
It means that views can be constructed also from a const registry and they
require to propagate the constness of the registry to the types used to
construct the views themselves:

```cpp
entt::view<const position, const velocity> view = std::as_const(registry).view<const position, const velocity>();
```

Consider the following definition for a non-const view instead:

```cpp
entt::view<position, const velocity> view = registry.view<position, const velocity>();
```

In the example above, `view` can be used to access either read-only or writable
`position` components while `velocity` components are read-only in all
cases.<br/>
In other terms, these statements are all valid:

```cpp
position &pos = view.get<position>(entity);
const position &cpos = view.get<const position>(entity);
const velocity &cpos = view.get<const velocity>(entity);
std::tuple<position &, const velocity &> tup = view.get<position, const velocity>(entity);
std::tuple<const position &, const velocity &> ctup = view.get<const position, const velocity>(entity);
```

It's not possible to get non-const references to `velocity` components from the
same view instead and these will result in compilation errors:

```cpp
velocity &cpos = view.get<velocity>(entity);
std::tuple<position &, velocity &> tup = view.get<position, velocity>(entity);
std::tuple<const position &, velocity &> ctup = view.get<const position, velocity>(entity);
```

Similarly, the `each` member functions will propagate constness to the type of
the components returned during iterations:

```cpp
view.each([](const auto entity, position &pos, const velocity &vel) {
    // ...
});
```

Obviously, a caller can still refer to the `position` components through a const
reference because of the rules of the language that fortunately already allow
it.

## Give me everything

Views are narrow windows on the entire list of entities. They work by filtering
entities according to their components.<br/>
In some cases there may be the need to iterate all the entities still in use
regardless of their components. The registry offers a specific member function
to do that:

```cpp
registry.each([](auto entity) {
    // ...
});
```

It returns to the caller all the entities that are still in use by means of the
given function.<br/>
As a rule of thumb, consider using a view if the goal is to iterate entities
that have a determinate set of components. A view is usually much faster than
combining this function with a bunch of custom tests.<br/>
In all the other cases, this is the way to go.

There exists also another member function to use to retrieve orphans. An orphan
is an entity that is still in use and has no assigned components.<br/>
The signature of the function is the same of `each`:

```cpp
registry.orphans([](auto entity) {
    // ...
});
```

To test the _orphanity_ of a single entity, use the member function `orphan`
instead. It accepts a valid entity identifer as an argument and returns true in
case the entity is an orphan, false otherwise.

In general, all these functions can result in poor performance.<br/>
`each` is fairly slow because of some checks it performs on each and every
entity. For similar reasons, `orphans` can be even slower. Both functions should
not be used frequently to avoid the risk of a performance hit.

# Iterations: what is allowed and what is not

Most of the _ECS_ available out there have some annoying limitations (at least
from my point of view): entities and components cannot be created nor destroyed
during iterations.<br/>
`EnTT` partially solves the problem with a few limitations:

* Creating entities and components is allowed during iterations.
* Deleting an entity or removing its components is allowed during iterations if
  it's the one currently returned by the view. For all the other entities,
  destroying them or removing their components isn't allowed and it can result
  in undefined behavior.

Iterators are invalidated and the behavior is undefined if an entity is modified
or destroyed and it's not the one currently returned by the view nor a newly
created one.<br/>
To work around it, possible approaches are:

* Store aside the entities and the components to be removed and perform the
  operations at the end of the iteration.
* Mark entities and components with a proper tag component that indicates they
  must be purged, then perform a second iteration to clean them up one by one.

A notable side effect of this feature is that the number of required allocations
is further reduced in most of the cases.

# Multithreading

In general, the entire registry isn't thread safe as it is. Thread safety isn't
something that users should want out of the box for several reasons. Just to
mention one of them: performance.<br/>
Views and consequently the approach adopted by `EnTT` are the great exception to
the rule. It's true that views and thus their iterators aren't thread safe by
themselves. Because of this users shouldn't try to iterate a set of components
and modify the same set concurrently. However:

* As long as a thread iterates the entities that have the component `X` or
  assign and removes that component from a set of entities, another thread can
  safely do the same with components `Y` and `Z` and everything will work like a
  charm. As a trivial example, users can freely execute the rendering system and
  iterate the renderable entities while updating a physic component concurrently
  on a separate thread.

* Similarly, a single set of components can be iterated by multiple threads as
  long as the components are neither assigned nor removed in the meantime. In
  other words, a hypothetical movement system can start multiple threads, each
  of which will access the components that carry information about velocity and
  position for its entities.

This kind of entity-component systems can be used in single threaded
applications as well as along with async stuff or multiple threads. Moreover,
typical thread based models for _ECS_ don't require a fully thread safe registry
to work. Actually, users can reach the goal with the registry as it is while
working with most of the common models.

Because of the few reasons mentioned above and many others not mentioned, users
are completely responsible for synchronization whether required. On the other
hand, they could get away with it without having to resort to particular
expedients.

## Views and Iterators

A special mention is needed for the iterators returned by the views. Most of the
time they meet the requirements of **random access iterators**, in all cases
they meet at least the requirements of **forward iterators**.<br/>
In other terms, they are suitable for use with the **parallel algorithms** of
the standard library. If it's not clear, this is a great thing.

As an example, this kind of iterators can be used in combination with
`std::for_each` and `std::execution::par` to parallelize the visit and therefore
the update of the components returned by a view, as long as the constraints
previously discussed are respected.<br/>
This can increase the throughput considerably, even without resorting to who
knows what artifacts that are difficult to maintain over time.
