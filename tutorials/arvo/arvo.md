+++
title = "Arvo"
weight = 1
template = "doc.html"
aliases = ["/docs/learn/arvo/arvo/"]
+++
Arvo, also called Urbit OS, is our operating system.

# Introduction

In this section we give a summary of the content and cover prerequisites for understanding the material presented here.

## Overview

This article is intended to provide a thorough summary of all of the most important aspects of the Arvo kernel. We work on two levels - a conceptual level that should be useful to anybody interested in how Arvo works, and a more technical level intended for those that intend to write software for Urbit.

Unlike any other popular operating system, it is possible for a single human to
understand every aspect of Arvo due to its compact size. The entire Urbit stack
is around 30k lines of code, while the Arvo kernel is just 1k lines of code. We
strive for a small code base because the difficulty in administering a system is
roughly proportional to the size of its code base.

We include a number of quotes from the Urbit [white paper](https://media.urbit.org/whitepaper.pdf) where appropriate. The white paper is a good companion to this article, though it should be noted that some parts of it are now either out of date or not yet implemented.

### Prerequisites

The conceptual level may mostly be understood without knowing Hoon, but some of it will require understanding at the level of Chapter One of the [Hoon tutorial](@/docs/tutorials/hoon/_index.md). The technical level will require Chapter One for full understanding, and some material from Chapter Two will be helpful as well. At the bare minimum, we presume that the reader has read through the [Technical Overview](@/docs/concepts/technical-overview.md), though some of the content presented there is presented again here in greater detail.

We also suggest to the reader to peruse the [glossary](@/docs/glossary/_index.md) before diving into this article. It will provide the initial scaffolding that you will be able to gradually fill in as you read this article and go deeper into the alternate universe of computing that is Urbit.

### Table of Contents
We present here a brief summary of each section.

#### [What is Arvo?](#what-is-arvo)
The big picture of Arvo.

#### [The stack](#the-stack)
An overview of each layer of the Arvo stack and how they interact, from Unix to userspace.

#### [The kernel](#the-kernel)

A description of how the Arvo kernel functions, including the basic arms and the structure of the event log.

#### [Vanes](#vanes)
A short description of each Arvo kernel module, known as a vane.

#### [File system](#file-system)
Essential information about Clay, our file system vane.

#### [Boot sequence](#boot-sequence)
An annotation of what is printed on the screen when you boot your ship for the first time, as well as subsequent boots.

#### [Security](#security)
Arvo's security features, including cryptography and protection against Sybil attacks.

# Consider splitting this into a different page. Basically the above is "Arvo as if it were on bare metal", and the rest is "how Arvo gets turned into a mess of Unix events"

#### [Virtual machine](#virtual-machine)
How the Nock runtime environment and virtual machine which Arvo lives is in implemented.

#### [Jets](#jets)
How Nock is made to run quickly.

#### [The worker and the daemon](#the-worker-and-the-daemon)
How Arvo is split into a worker and a daemon.

#### I/O
How I/O is handled by Arvo.

# What is Arvo?

Arvo is a [non-preemptive](https://en.wikipedia.org/wiki/Cooperative_multitasking) operating system purposefully built to create a new internet whereby users own and manage their own data. Despite being an operating system, Arvo does not replace Windows, Mac OS, or Linux. It is better to think of the user experience of Arvo as being closer to that of a web browser for a more human internet. As such, Arvo is generally run inside a virtual machine, though in theory it could be run on bare metal.

> Urbit is a “browser for the server side”; it replaces multiple developer-hosted web services on multiple foreign servers, with multiple self-hosted applications on one personal server.

Every architectural decision for Arvo (and indeed, the entire Urbit stack) was made with this singular goal in mind. Throughout this document, we connect the various concepts underlying Arvo with this overarching goal.




---the following 2 paragraphs was copied from the old doc, probably ought to be put somewhere else.

At a high level Arvo takes a mess of Unix I/O events and turns them into something clean and structured for the programmer.

Arvo is designed to avoid the usual state of complex event networks: event spaghetti. We keep track of every event's cause so that we have a clear causal chain for every computation. At the bottom of every chain is a Unix I/O event, such as a network request, terminal input, file sync, or timer event. We push every step in the path the request takes onto the chain until we get to the terminal cause of the computation. Then we use this causal stack to route results back to the caller.


## An operating function

Arvo is the world's first _purely functional_ operating system, and as such it may reasonably be called an _operating function_. The importance of understanding this design choice and its relevence to the overarching goal cannot be understated. If you knew only a single thing about Arvo, let it be this.

This means two things: one is that the there is a notion of _state_ for the operating system, and that the current state is a pure function of the [event log](#event-log), which is a chronological record of every action the operating system has ever performed. A _pure function_ is a function that always produces the same output given the same input. Another way to put this is to say that Arvo is _deterministic_. Other operating systems are not deterministic for a number of reasons, but one simple reasons is because they allow programs to alter global variables that then affect the operation of other programs.
 
In mathematical terms, one may think of Arvo as being given by a transition function _T_:

```
T: (State, Input) -> (State, Output).
```

In practice, _T_ is implemented by the `+poke` arm of the Arvo kernel, which is described in more detail in the [kernel section](#the-kernel). In theoretical terms, it may be more practical to think of Arvo as being defined by a _lifecycle function_ we denote here by _L_:

```
L: History -> State.
```

Which perspective is more fruitful depends on the problem being considered.

### Deterministic

We consider Arvo to be deterministic at a high level. By that we mean that it is
stacked on top of a frozen instruction set known as Nock. Frozen instruction
sets are a new idea for an operating system, but not for computing in general.
For instance, the CPU instruction sets such as
[x86-64](https://en.wikipedia.org/wiki/X86-64v) are frozen at the level of the
chip. A given operating system may be adapted to run on more than one CPU
instruction set, we merely freeze the instruction set at a higher level in order
to enable deterministic computation.

Arvo handles nondeterminism in an interesting way. Deciding whether or not to
halt a computation that could potentially last forever becomes a heuristic
decision that is akin to dropping a packet. Thus it behooves one to think of
Arvo as being a packet transceiver rather than a full fledged computer - events
are never guaranteed to complete.

Because Arvo is run on a VM, nondeterministic information such as the stack
trace of an infinite loop that was entered into may be obtained. This is
possible because while Arvo may be unable to obtain that information, the
interpreter beneath does and can inject that information back into the event log.

Being deterministic at a high level enables many
things that are out of reach of any other operating system. For
instance, we are able to do [over the air](#over-the-air-updates) (OTA) updates,
which allows bug fixes to be implemented across the network without needing to
worry whether it won't work on someone's ship, since Arvo is an
[interpreter](#solid-state-intrepeter) that can accept source code to
update itself instead of requiring a pre-compiled binary. This essential
property is why Urbit is able to act as a personal server while only having the
complexity to use of a web browser.

### Event log

>The formal state of an Arvo instance is an event history, as a linked list of nouns from first to last. The history starts with a bootstrap sequence that delivers Arvo itself, first as an inscrutable kernel, then as the selfcompiling source for that kernel. After booting, we break symmetry by delivering identity and entropy. The rest of the log is actual input.

The Arvo event log is a list of every action ever performed on your ship that lead up to the current state. In principle, this event log is maintained by the [Nock runtime environment](#virtual-machine), but in practice event logs become too long over time to keep. Thus periodic snapshots of the state of Arvo are taken and the log up to that state is pruned.

The beginning of the event log starting from the very first time a ship is
booted up until the kernel is compiled and identity and entropy are created is a
special portion of the Arvo lifecycle known as the _larval stage_. We describe
the larval stage in more detail in the [larval stage](#larval-stage-core) section.

More information on the structure of the Arvo event log and the Arvo state is given in the section on [the kernel](#the-kernel).

>User-level code is virtualized within a Nock interpreter written in Hoon (with zero virtualization overhead, thanks to a jet). Arvo defines a typed, global, referentially transparent namespace with the Ames network identity (page 34) at the root of the path. User-level code has an extended Nock operator that dereferences this namespace and blocks until results are available. So the Hoon programmer can use any data in the Urbit universe as a typed constant.

> Most Urbit data is in Clay, a distributed revision-control vane. Clay is like a typed Git. If we know the format of an object, we can do a much better job of diffing and patching files. Clay is also good at subscribing to updates and maintaining one-way or two-way automatic synchronization.


### Over the air updates

> Nock is frozen, but Arvo can hotpatch any other semantics at any layer in the
> system (apps, vanes, Arvo or Hoon itself) with automatic over-the-air updates.

Typically updates to an operating system are given via a pre-compiled binary,
which is why some updates will work on some systems but not on others where the
environment may differ. This is not so on Arvo - because it is an
[interpreter](#solid-state-interpreter), Arvo may update itself by receiving
source code from your sponsor over [Ames](@/docs/tutorials/arvo/ames.md), our
network.

### Solid state interpreter

>We call an execution platform with these three properties (universal persistence, source-independent packet I/O, and high-level determinism) a solid-state interpreter (SSI). A solid-state interpreter is a stateful packet transceiver. Imagine it as a chip. Plug this chip into power and network; packets go in and out, sometimes changing its state. The chip never loses data and has no concept of a reboot; every packet is an ACID transaction.

We call Arvo a _solid state interpreter_. In this section we describe what is meant by this, and how this behavior derives from the fact that Arvo is an [ACID database](#acid-database) and a [single-level store](#single-level-store).

In computer science, an _interpreter_ is a program that directly executes
instructions written in some human-understandable programming or scripting
language, rather than requiring the code to first be compiled into a machine
language. Arvo is an interpreter, which is important for us since it allows us
to perform derministic [over-the-air updates](#over-the-air-updates) by the
direct transfer of raw source code.

To understand what we mean by _solid state_ interpreter, consider the operation of a solid state hard drive when a computer shuts down or loses power. Data written to an SSD is permanent unless otherwise deleted - loss of power may leave some partially written data, but nothing is ever lost. Thus, the state of an SDD can be considered to be equivalent to the data that it contains. That is to say, you do not need to know anything about the system which is utilizing the SSD to know everything there is to know about the SSD. There is no notion of "rebooting" a SSD - it simply stores data, and when power is restored to it, it is in exactly the same state as it was when power was lost.

Contrast this with other popular operating systems, such as Windows, Mac, or Linux. The state of the operating system is something that crucially depends on having a constant power supply because much of the state of the operating system is stored in RAM, which is volatile. When you reboot your computer or suddenly lose power, any information stored in RAM is lost. Modern operating systems do mitigate this loss of information to some extent. For instance, it may remember what applications you were running at the time power was lost and try to restore them. Particularly durable programs may go as far as writing their state to disk every few seconds so that only very minimal information can be lost in a power outage. However, this is not the default behavior, and indeed if it were then they would be so slow as to be unusable, as you would effectively be using your hard disk as the RAM.

How Arvo handles loss of power is closer to that of an SSD. Since it is an [ACID database](#acid-database) and a [single-level store](#single-level-store), a sudden loss of power have no effect on the state of the operating system. When you boot your ship back up it will be exactly as it was before the failure. Your information can never be lost, and because it was designed from the ground up to behave in this fashion, it does not suffer significant slowdown by persisting all data in this manner as would be the case if your typical operating system utilized its hard disk as RAM.

#### ACID Database
In the client-server model, data is stored on the server and thus reliable and efficient databases are an integral part of server architecture. This is not quite so true for the client - a user may be expected to reboot their machine in the middle of a computation, alter or destroy their data, never make backups or perform version control, etc. In other words, client systems like your personal computer or smart phone are not well suited to act as databases.

In order to dismantle the client-server model and build a peer-to-peer internet, we need the robustness of modern database servers in the hands of the users. Thus the Arvo operating system itself must have all of the properties of a reliable database.

Database theory studies in precise terms the possible properties of anything that could be considered to be a database. In this context, Arvo has the properties of an [ACID database](https://en.wikipedia.org/wiki/ACID), and the Ames network could be thought of as network of such databases. ACID stands for _atomicity_, _consistency_, _isolation_, and _durability_. We review here how Arvo satisfies these properties.

 - Atomicity: Events in Arvo are _atomic_, meaning that they either succeed completely or fail completely. In other words, there are no transient periods in which something like a power failure will leave the operating system in an invalid state. When an event occurs in Arvo, e.g. [the kernel](#the-kernel) is `poke`d, the effect of an event is computed, it is [persisted](https://en.wikipedia.org/wiki/Persistence_(computer_science)) by writing it to the event log, and only then is the actual event applied.

 - Consistency: Every possible update to the database puts it into another valid state. Given that Arvo is purely functional, this is easier to accomplish than it would be in an imperative setting.

 - Isolation: Transactions in databases often happen concurrently, and isolation ensures that the transactions occur as if they were performed sequentially, making it so that their effects are isolated from one another. Arvo ensures this simply by the fact that it only ever performs events sequentially. Arvo transactions cannot be considered _entirely_ sequential though, as the [worker and daemon](#the-worker-and-the-daemon) operate in parallel.
 
 - Durability: Completed transactions will survive permanently. This is one way in which Arvo greatly differs from other operating systems - there is no way to truly delete a file, rather you can just mark it as being deleted and have the file effectievly be ignored. This is due to the structure of our file system known as [Clay](@/docs/tutorials/arvo/clay.md), which is entirely version controlled. That is to say, every version of every file remains on your system forever.
 
 Durability is a necessary consequence of Arvo's [referential transparency](https://en.wikipedia.org/wiki/Referential_transparency), which means that you can always replace an expression by what it evaluates to without changing its behavior. For example, you can always replace a file referred to in a program by the contents of that file, because you know that file is never going to change (or rather, if it does, the old version will still be accessible).

#### Single-level store

>A kernel which presents the abstraction of a single layer of permanent state is also called a single-level store. One way to describe a single-level store is that it never reboots; a formal model of the system does not contain an operation which unpredictably erases half its brain.

Today's operating systems utilize at least two types of memory: the hard disk and the RAM, and this split is responsible for the fact that data is lost whenever power is lost. Not every operating system in history was designed this way - in particular, [Multics](https://en.wikipedia.org/wiki/Multics) utilized only one store of memory. Arvo takes after Multics - all data is stored in one permanent location, and as a result no data is ever lost when power is lost.

### Event-driven

### Non-preemptive


### Interacting with Arvo

Not sure we need this section

#### Dojo

You will first interact with your instance of Arvo with a [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) known as the dojo. The dojo allows you to input commands to your ship as well as run Hoon code. You can find a comprehensive guide to using your ship in dojo in our guide on [using your ship](@/using/operations/using-your-ship.md)y.

(picture of dojo)

## The stack

<img class="mv5 w-100" src="https://media.urbit.org/docs/arvo/stack.png">

Ultimately, everything that happens in Arvo is reduced to Unix events. Here we depict the stack from userspace apps Dojo and Landscape down to Unix. (I need to change this, Landscape is not actuall a Gall app)

## The kernel

The Arvo kernel, stored in `sys/arvo.hoon`, is about 1k lines of Hoon whose primary purpose is to implement the transition function, `+poke`. In this section we point out the most important parts of `arvo.hoon` and describe their role in the greater system. We also give brief descriptions of Arvo's kernel modules, known as vanes, and how Arvo interfaces with them.

This section requires an understanding of Hoon of at least the level of Chapter One of the [Hoon tutorial](@/docs/tutorials/hoon/_index.md).

### Overall structure

`arvo.hoon` contains five top level cores as well as a "formal interface" consisting of a single gate that implements the transition function. They are nested with the `=<` and `=>` runes like so, where items lower on the list are contained within items higher on the list:
 + Types
 + Section 3bE Arvo Core
 + Implementation core
 + Structural interface core, or adult core
 + Larval stage core
 + Formal interface
 
See [Section 1.7](@/docs/tutorials/hoon/arms-and-cores/#core-nesting) of the Hoon tutorial for further explanation of what is meant here by "nesting". We now describe the functionality of each of these components.
 
#### Formal interface

The formal interface is a single gate that takes in the current time and a noun that encodes the input. This input, referred to as an _event_, is then put into action by the `+poke` arm, and a new noun denoting the current [state of Arvo](#the-state) is returned. In reality, you cannot feed the gate just any noun - it will end up being an `ovum` described below - but as this is the outermost interface of the kernel the types defined in the type core are not visible to the formal interface.

```hoon
    ::  Arvo formal interface
    ::
    ::    this lifecycle wrapper makes the arvo door (multi-armed core)
    ::    look like a gate (function or single-armed core), to fit
    ::    urbit's formal lifecycle function. a practical interpreter
    ::    can ignore it.
    ::
    |=  [now=@da ovo=*]
    ^-  *
    ~>  %slog.[0 leaf+"arvo-event"]
    .(+> +:(poke now ovo))
```
 
#### Types

This core contains the most basic types utilized in Arvo. We discuss a number of them here.

##### `+duct`

The `Arvo` causal stack is called a `duct`. This is represented simply as a list of paths, where each path represents a step in the causal chain. The first element in the path is the first letter of whichever vane handled that step in the computation, or the empty span for Unix.

Here's a `duct` that was recently observed in the wild (I should redo this to make sure its up to date)

```
~[
  /g/a/~zod/4_shell_terminal/u/time
  /g/a/~zod/shell_terminal/u/child/4/main
  /g/a/~zod/terminal/u/txt
  /d/term-mess
  //term/1
]
```

<img class="mv5 w-100" src="https://media.urbit.org/docs/arvo/timer.png">

This is the duct the timer vane, Behn, receives when the "timer" sample app asks the timer vane to set a timer. This is also the duct over which the response is produced at the specified time. Unix sent a terminal keystroke event (enter), and Arvo routed it to Dill (our terminal), which passed it on to the Gall app terminal, which sent it to shell, its child, which created a new child (with process id 4), which on startup asked Behn to set a timer.

Behn saves this duct, so that when the specified time arrives and Unix sends a wakeup event to the timer vane, it can produce the response on the same duct. This response is routed to the place we popped off the top of the duct, i.e. the time app. This app produces the text "ding", which falls down to the shell, which drops it through to the terminal. Terminal drops this down to Dill, which converts it into an effect that Unix will recognize as a request to print "ding" to the screen. When dill produces this, the last path in the duct has an
initial element of the empty span, so this is routed to Unix, which applies the effects.

This is a call stack, with a crucial feature: the stack is a first-class citizen. You can respond over a duct zero, one, or many times. You can save ducts for later use. There are definitely parallels to Scheme-style continuations, but simpler and with more structure.


##### `wire`

Synonym for path, used in ducts.

##### `move`

If ducts are a call stack, then how do we make calls and produce results? Arvo processes `moves` which are a combination of message data and metadata. There are two types of `move`s. A `%pass` move is analogous to a call:

```
[duct %pass return-path=path vane-name=@tD data=card]
```

Arvo pushes the return path (preceded by the first letter of the vane name) onto the duct and sends the given data, a card, to the vane we specified. Any response will come along the same duct with the `wire` `return-path`.

A `%give` `move` is analogous to a return:

```
[duct %give data=card]
```

Arvo pops the top `wire` off the duct and sends the given card back to the caller.

##### `card`

Note: this section seems out of date, there is no longer a `+card` in `arvo.hoon`, and `move`s there consist of an `arvo` and a `duct`. An `arvo` is described to be an "Arvo card". hmm.

Cards are the vane-specific portion of a move. Each vane defines a protocol for interacting with other vanes (via Arvo) by defining four types of cards: tasks, gifts, notes, and signs.

When one vane is `%pass`ed a card in its `task` (defined in zuse), Arvo activates the `+call` gate with the card as its argument. To produce a result, the vane `%give`s one of the cards defined in its `gift`. If the vane needs to request something of another vane, it `%pass`es it a `note` card. When that other vane returns a result, Arvo activates the `+take` gate of the initial vane with one of the cards defined in its `sign`.

In other words, there are only four ways of seeing a move: (1) as a request seen by the caller, which is a `note`. (2) that same request as seen by the callee, a `task`. (3) the response to that first request as seen by the callee, a `gift`. (4) the response to the first request as seen by the caller, a `sign`.

When a `task` `card` is `%pass`ed to a vane, Arvo calls its `+call` gate, passing it both the card and its duct. This gate must be defined in every vane. It produces two things in the following order: a list of moves and a possibly modified copy of its context. The moves are used to interact with other vanes, while the new context allows the vane to save its state. The next time Arvo activates the vane it will have this context as its subject.

This overview has detailed how to pass a card to a particular vane. To see the cards each vane can be `%pass`ed as a `task` or return as a `gift` (as well as the semantics tied to them), each vane's public interface is explained in detail in its respective overview.

##### ovum

A pair of a `wire` and a `curd`, with a `curd` being like a typeless `card`. The reason for a typeless `card` is that this is the data structure which Arvo uses to communicate with the runtime, and Unix events have no type. Then the `wire` here is (always?) the default Unix `wire`, namely `//`. In particular, it is not a `duct` because `ovum`s come from the runtime rather than from within Arvo.

#### Arvo cores

`arvo.hoon` has four additional cores that encode the functionality of Arvo. The [larval core](#larval-stage-core), [structural interface core](#structural-interface-core), and [implementation core](#implementation-core) each have five arms, called `+come`, `+load`, `+peek`, `+poke`, and `+wish`. Of these five arms, only `+poke` affects the [Arvo state](#the-state), while the rest leave the state invariant. Thus `+poke` is the aforementioned transition function that sends Arvo from one state to the next.

A short summary of the purpose of each these arms are as follows:

 - `+poke` is the transition function that moves Arvo from one state to the
   next. It is the most fundamental arm in the entire system. It is a typed
   transactional message that is delivered exactly once. This is sometimes said
   to be impossible, but that is not the case for single-level stores engaged in
   a permanent session, as is the case among Arvo ships.
 - `+peek` is an arm used for inspecting things outside of the kernel. It grants
   read-only access to `scry` Arvo's global referentially transparent namespace.
 - `+wish` is a function that takes in a core and then parses and compiles it
   with the standard library, `zuse`. It is useful from the outside if you ever
   want to run code within. One particular way in which it is used is by the
   runtime to read out the version of `zuse` so that it knows if it is
   compatible with this particular version of the kernel.
 - `+load` is used when upgrading the kernel. It is only ever called by Arvo
   itself, never by the runtime. If upgrading to a kernel where types are
   compatible, `+load` is used, otherwise `+come` is used.
 - `+come` is used when the new kernel has incompatible types, but ultimately
 reduces to a series of `+load` calls.
 
 The [Section 3bE core](#section-3be-core) does not follow this pattern.

##### Section 3bE core

This core defines helper functions that are called in the larval and adult cores. These helper functions are placed here for safety so that they do not have access to the entire [state of Arvo](#the-state), which is contained in the structural interface core. One reason this core is required is that the Arvo interface has only five arms and additional arms are required to perform everything the kernel needs to do in a clean manner, so they must be segregated from the other three cores that stick to the five-arm paradigm.

##### Implementation core

This core is where the real legwork of the Arvo kernel is performed during the adult stage. It does not communicate with Unix directly, rather it is called by the structural interface core.

##### Structural interface core

This core could be thought of as the primary "adult core" - the one that is in operation for the majority of its lifecycle and the one that contains the [Arvo state](#the-state). This core should be thought of as an interface - that is, the amount of work the code does here is minimal as its main purpose is to be the core that communicates with Unix, and Unix should not be able to access deeper functions stored in the implementation core and 3bE core on its own. Thus, arms in this core are there primarily to call arms in the implementation core and 3bE core.

##### Larval stage core

This core is in use only during the larval stage of Arvo, which is after the Arvo kernel has compiled itself but before it has "broken symmetry" by acquiring identity and entropy, the point at which the larval stage has concluded. We call this breaking symmetry because prior to this point, every Urbit is completely identical. The larval stage performs the following steps in order:

 + The standard library, `zuse`, is installed.
 + Entropy is added
 + Identity is added
 + Metamorph into the next stage of Arvo
 
Once the larval stage has passed its functionality will never be used again.

### The state

The Arvo transition function, called `+poke`,  takes the current state of Arvo and an event and outputs the new state of Arvo and the response to the event, if any. Here we explain how the state of Arvo is structured.

```hoon
::  persistent arvo state
::
=/  pit=vase  !>(..is)                                  ::
=/  vil=vile  (viol p.pit)                              ::  cached reflexives
=|  $:  lac=_&                                          ::  laconic bit
        eny=@                                           ::  entropy
        our=ship                                        ::  identity
        bud=vase                                        ::  %zuse
        vanes=(list [label=@tas =vane])                 ::  modules
    ==                                                  ::
```

```hoon
=/  pit=vase  !>(..is)                                  ::
```
This `vase` is part of the state but does not get directly migrated when `+poke`
is called. `!>(..is)` consists of the code in `arvo.hoon` written above this core contained in
a `vase`. Thus this part of the state changes only when that code changes in an
update.

```hoon
=/  vil=vile  (viol p.pit)                              ::  cached reflexives

```
This is a cache of specific types that are of fundamental importance to Arvo -
namely `type`s, `duct`s, `path`s, and `vase`s. This is kept because it is
unnecessarily wasteful to recompile these fundamental types on a regular basis.
Again, this part of the state is never updated directly by `+poke`.

```hoon
=|  $:  lac=_&                                          ::  laconic bit
        eny=@                                           ::  entropy
        our=ship                                        ::  identity
        bud=vase                                        ::  %zuse
        vanes=(list [label=@tas =vane])                 ::  modules
    ==
```
This is where the real state of the Arvo kernel is kept. `lac` detemines whether
Arvo's output is verbose, which can be set using the `|verb` command in the
dojo. `eny` is the current entropy. `our` is the ship, which is permanently
frozen during the larval stage. `bud` is the standard library. Lastly, `vanes`
is of course the list of vanes, which have their own internal states. As Hoon is
homoiconic, code is data, so this is does secrely have everything the vanes are
doing encoded within, including all user apps. 

As you can see, the state of Arvo is quite simple. It's primary role is that of
a traffic cop.


## Vanes

The Arvo kernel can do very little on its own. Its functionality is extended in a careful and controlled way with vanes, also known as kernel modules.

As described above, we use Arvo proper to route and control the flow of `move`s. However, Arvo proper is rarely directly responsible for processing the event data that directly causes the desired outcome of a move. This event data is contained within a `card`. Instead, Arvo proper passes the `card` off to one of its vanes, which each present an interface to clients for a particular well-defined, stable, and general-purpose piece of functionality.

As of this writing, we have nine vanes, which each provide the following services:

- `Ames`: the name of both our network and the vane that communicates over it.
- `Behn`: a simple timer.
- `Clay`: our version-controlled, referentially- transparent, and global filesystem.
- `Dill`: a terminal driver. Unix sends keyboard events to `%dill` from either the console or telnet, and `%dill` produces terminal output.
- `Eyre`: an http server. Unix sends http messages to `%eyre`, and `%eyre` produces http messages in response.
- `Ford`: handles resources and publishing.
- `Gall`: manages our userspace applications. `%gall` keeps state and manages subscribers.
- `Iris`: an http client.
- `Jael`: storage for Azimuth information. 

## Boot sequence

### Larval stage

>Before we plug the newborn node into the network, we feed it a series of bootstrap or “larval” packets that prepare it for adult life as a packet transceiver on the public network. The larval sequence is private, solving the secret delivery problem, and can contain as much code as we like.


## Security

> A new stack, designed as a unit, learning from all the mistakes of 20th-century software and repeating none of them, should be simpler and more rigorous than what we use now. Otherwise, why bother?

### Sybil attacks

[https://en.wikipedia.org/wiki/Sybil_attack](Sybil attacks) are when a malicious actor creates a large number of identites on a network in order to gain a disproportionate influence on the network. This could be used to do things such as rig a voting scheme or perform a denial-of-service attack, which is where a node is spammed with spurious requests rendering it unable to perform its ordinary functions. Any distributed system needs protection against such an attack, and Urbit is no different.

One common question one may have when first encountering Urbit is why there is a limited address space. Even more perplexing is the fact that the number of planets is less than the number of humans there are on Earth. We explain here why this choice is an effective preventative measure against Sybil attacks. 

If a malicious actor is able to create an unlimited number of identities that appear to be legitimate people (i.e., planets), they can easily perform a Sybil attack. We prevent this by making planets scarce - there are fewer planets than there are humans, and acquiring one has some sort of cost associated with it. The expected gain of a Sybil attack would have to be very high to offset this cost.

Once a Sybil attack is recognized the node under attack is now in possession of cryptographic evidence of the attack as all of the messages that the attack is composed of are signed by those planets. They can then proceed to block all traffic from these planets, and could prove to other users of the network that these planets are untrustworthy with the evidence acquired during the attack. Thus the malicious actor's resources are thus exhausted in the attack - they cannot then turn around and perform another one on the same node without acquiring more planets, and anybody who the attackee has shared their blacklist with will also be immune to attack.

Furthermore, a reasonable scenario would be one in which a malicious actor possesses a star and is using that star to generate all of the planets. This would certainly be faster than trying to purchase thousands of planets from star holders individually. Thus when it becomes evident that all planets that are part of the attack originated from a certain star, it would be easy to simply blacklist every planet created by that star. Obviously that would also run the risk of blacklisting legitimate users that the malicious actor sold a planet to - that would have to be handled on a case by case basis, but one can imagine that future Urbit ID marketplaces will have a reputation system that should prevent this from happening in many circumstances.

Thus permanent identity is a crucial component of the ``immune system'' for the network, and gossiping of blacklists is how immunity spreads through the network.

What about smaller ships, such as moons or comets? There are so many possible comets that they may as well be infinite, and this is where the hierarchical nature of the network acts to our benefit. When a human wishes to be recognized as a human, they would be expected to use a planet. To anybody who believes they could potentially be the target of a denial of service attack, they can simply ignore traffic from comets across the board. If someone needs to register additional ships beyond their planet at some web service, they could use their planet to vouch for their moons/comets.
