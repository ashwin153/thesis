---
abstract: |
    Concurrency is *hard*. Concurrency refers to situations in which
    multiple programs simultaneously modify shared data. Concurrent programs
    may be run across threads, processes, and, in the case of distributed
    systems, networks. Concurrency is challenging because it introduces
    ambiguity in execution order, and it is precisely this ambiguity that
    cause a class of failures known as race conditions. Race conditions
    manifest themselves in subtle ways in concurrent systems, and they can
    often be difficult to detect and challenging to remove.

    Many programming languages provide fundamental abstractions such as
    locks, semaphores, and monitors to explicitly deal with race conditions.
    Some, like Rust [@rust], go a step further and are able to statically
    detect race conditions between concurrent threads. But none, however,
    are able guarantee that race conditions will be completely eliminated
    from a distributed system. Distributed systems form the computing
    backbone of nearly every major technology from social networks to video
    streaming, but their intricate complexity coupled with the inability to
    detect race conditions makes designing them extremely error-prone.

    In this thesis, we will explore transactional programming and its
    application to building race-free distributed systems. We will first
    discuss Caustic, a transactional programming language that allows
    programs to be safely distributed without any explicit synchronization
    or coordination. We will show that Caustic requires the underlying
    storage system to satisfy a minimal contract. We will then discuss
    Beaker, a distributed, fault-tolerant database that efficiently
    implements this contract.
author:
- |
    Ashwin Madavan\
    `ashwin.madavan@gmail.com`
- 'Christopher J. Rossbach\'
bibliography:
- 'caustic/caustic.bib'
- 'beaker/beaker.bib'
title: On Transactional Programming
---

\maketitle
\tableofcontents
Caustic: A Transactional Programming Language
=============================================

In this chapter, we will discuss Caustic. Caustic is a transactional
programming language that allows programs to be written as if they were
single-threaded applications, but permits them to be distributed
arbitrarily without error. We first motivate transactional programming
by discussing concurrency and the various techniques that are typically
used to deal with it. We then discuss the various components of Caustic:
a **runtime** that executes programs, a **standard library** that
simplifies program construction, and a **compiler** that simplifies
program expression. We conclude with an evaluation of the
programmability and performance of the language.

Concurrency
-----------

Concurrency is *hard*. Concurrency refers to situations in which
multiple programs simultaneously modify shared data. Concurrent programs
may be run across threads, processes, and, in the case of distributed
systems, networks. Concurrency is challenging because it introduces
ambiguity in execution order, and it is precisely this ambiguity that
causes a class of failures known as race conditions.

Race conditions may occur whenever the order in which concurrent
programs are executed affects their outcome. For example, suppose there
exist two programs $A$ and $B$ that each increment a shared counter $x$.
Formally, each program reads the current value of $x$ and then writes
$x + 1$. If $B$ reads *after* $A$ writes, then $B$ reads $x + 1$ and
writes $x + 2$. However, if $B$ reads *before* $A$ writes but after $A$
reads, then both $A$ and $B$ will read $x$ and write $x + 1$. This is an
example of a race condition, because the value of the counter $x$ after
both $A$ and $B$ have completed depends on the order in which $A$ and
$B$ are executed. This race condition may seem relatively benign, but it
can have catastrophic consequences in practical systems. Suppose the
value of $x$ corresponded to your bank balance. What if your bank
determined your balance differently depending on the order in which
deposits are made? Race conditions manifest themselves in subtle ways in
concurrent systems, and they can often be difficult to detect and
challenging to remove.

Before introducing mechanisms to deal with race conditions, we must
first define the necessary and sufficient criteria that any such
mechanism must satisfy to correctly mitigate them. Race conditions are a
relatively well-studied problem in database literature, and we may draw
upon known results from the field to define four properties of race-free
programs. [@transactions] We will hereafter refer to anything that
satisfies these properties as **transactional**.

-   **Atomic**: Programs are all-or-nothing; they are never partially
    applied.

-   **Consistent**: Programs see the effect of all completed programs.

-   **Isolated**: Programs cannot see the effect of in-progress
    programs.

-   **Durable**: The effects of completed programs are permanently
    visible.

### Transactional Databases

A simple approach to dealing with race conditions is to use a
transactional database. Programs perform any accesses and modifications
to shared data within a transaction that is then executed on the
underlying database.

While transactional databases provide the necessary guarantees on which
correct distributed systems can be built, their collective lack of a
robust and uniform query language makes it all but impossible to design
non-trivial applications. Recent years have marked a proliferation in
NoSQL databases that scale well by shedding functionality. These
databases were not popularized because of their query languages, they
were *in spite* of them. Some, like Cassandra and Aerospike, attempt to
mimic the relational semantics of SQL, but they fall short of
implementing the entire SQL specification. Others, like MongoDB and
DynamoDB, implement entirely new query languages. Relational databases
like MySQL and PostgreSQL that each claim to implement the same SQL
specification actually support incompatible subsets of its
functionality. Christopher Date, one of the fathers of relational
databases, criticized SQL for its lack of a canonical implementation and
its ambiguous and often unintuitive syntax. [@sql] Stark differences
between query languages tightly couples storage systems and the programs
that are run on them, and makes the choice of database in an application
effectively permanent. Programmers can recover some degree of
interoperability by using an object-relational mapping (ORM). However,
these ORMs often have unintuitive syntax and poor performance.

### Pessimistic Concurrency

Another way to deal with race conditions is pessimistic concurrency, or
**locking**. Before a program accesses or modifies shared data, it first
declares its intentions to other programs by acquiring a lock that it
subsequently releases when it is finished with the data. Locks are
trivially transactional because they require programs to have exclusive
ownership before performing any operations on shared data. However,
locking introduces new problems.

First, concurrent acquisition of multiple locks can cause a **deadlock**
that prevents the system from making progress. For example, if program
$A$ acquires lock $x$ and then attempts to acquire lock $y$ while
program $B$ acquires $y$ and then attempts to acquire $x$, then neither
$A$ nor $B$ can make progress because each is waiting for the other to
release their lock. This problem is typically mitigated by imposing a
total order on lock acquisition; for any two locks $x$ and $y$, $x$ will
always be acquired before $y$ or $y$ will always be acquired before $x$.

Second, faulty programs may never release their locks. For example, if
program $A$ acquires lock $x$ and subsequently fails, then no other
program can ever acquire $x$. This problem is typically mitigated by
using **leases**. [@leases] A lease is a lock that is automatically
released after a certain amount of time. Programs must be carefully
constructed so that they complete their operations on shared data within
the lease duration or refresh the lease before it expires.

Third, locking is expensive. Locks must be acquired regardless of
whether or not there actually are concurrent operations on shared data,
because programs cannot know if there are or will be other programs that
want or will want to simultaneously use the data. This significantly
degrades performance in situations where contention between programs is
low.

Fourth, locking cannot protect against programmer error. Programmers may
omit or incorrectly use a lock and thereby introduce race conditions
into their program. Locks cannot guarantee they will be used correctly,
and, therefore, cannot guarantee that race conditions will be
exhaustively eliminated from a program.

### Optimistic Concurrency

Optimistic concurrency allows multiple programs to simultaneously
access, but not modify, shared data without acquiring locks. Each
program locally buffers any modifications that it makes to shared data
and attempts to atomically apply them when it completes conditioned on
the data that it accessed remaining unchanged. If any data was modified,
the program retries. This conditional update, known as a multi-word
compare-and-swap, is known to be transactional and is widely used in a
number of software transactional memory systems [@stm] including
Sinfonia [@sinfonia] and Caustic. We will hereafter refer to this
operation as **cas** and its arguments as a **transaction**. Optimistic
concurrency assumes that contention between programs will be low,
because frequent retries can significantly degrade performance.

Optimistic concurrency requires a mechanism to detect that data has
changed. The approach taken in Caustic and in similar systems is to
uniquely identify data with a **key** and to associate each key with
**revision**, or versioned value, whose version is incremented each time
that its value is changed. We say that revisions $A$ and $B$ for a key
**conflict** if the version of $A$ is less than $B$. Note that conflict
is an asymmetric relation; if $A$ conflicts with $B$, then $B$ does not
conflict with $A$.

Optimistic concurrency also assumes the existence of an underlying
key-value store, hereafter referred to as a **volume**, that supports
**get** and **cas** which access and modify data respectively in the
manner described above. We will provide a distributed, fault-tolerant
implementation of a volume in Beaker. For now, we will assume that such
an implementation exists.

Runtime
-------

A simple implementation of optimistic concurrency might get each key
whenever its value is needed by a program. This approach is performant
for in-memory volumes, but does not scale well when volumes are located
on disk or across networks. Instead, a scalable implementation of
optimistic concurrency might get each key whenever its value is needed
or *will be needed* by a program. This is the approach taken in the
**runtime**.

The **runtime** is a virtual machine that dynamically translates
**programs** into transactions. A program is an abstract-syntax tree
that is composed of **literals** and **expressions**. A literal is a
scalar value of type $flag$, $real$, $text$, or $null$ which correspond
to $bool$, $double$, $string$, and $null$ respectively in most C-style
languages. An expression is a function that transforms literal arguments
into a literal result. Expressions may be chained together arbitrarily
to form complex programs. For example, $read(key)$ returns the value of
the specified key and $write(key, value)$ updates the specified key with
the specified value.
Table [\[table:expressions\]](#table:expressions){reference-type="ref"
reference="table:expressions"} enumerates the various expressions that
are natively supported by the runtime.

### Execution

The runtime uses iterative partial evaluation to gradually reduce
programs into a single literal result according to the following
procedure.

1.  **Fetch**: Get all keys that are read or written anywhere in the
    program that have not been fetched before and add the returned
    revisions to a local **snapshot**.

2.  **Evaluate**: Recursively replace all expressions with literal
    arguments with their corresponding literal result. For example,

    $$\begin{gathered}
              add(real(1), sub(real(0), real(2))) \rightarrow real(-1)
              \end{gathered}$$

    The result of all writes is saved to a local **buffer** and the
    result of all reads is the latest value of the key in the local
    buffer or snapshot. This ensures that reads will see the effect of
    all previous writes within the program.

3.  **Repeat**: Loop until the program is reduced to a single literal.
    Because all expressions with literal arguments return a literal
    result, all programs will eventually reduce to a single literal.

4.  **Commit**: Cas all keys in the local buffer conditioned on all
    revisions in the local snapshot. The transactional guarantees of cas
    imply that program execution is **serializable**. Serializability
    means that concurrent execution has the same effect as some
    sequential execution, and, therefore, that program execution will be
    robust against race conditions.

### Optimizations

First, execution is tail-recursive. This allows programs to be composed
of arbitrarily many nested expressions without overflowing the stack
frame during execution. It also allows the Scala compiler to
aggressively optimize execution into a tight loop.

Second, the runtime batches I/O. Reads are performed simultaneously
whenever possible and writes are buffered and simultaneously committed.
By batching I/O, the runtime performs a minimal number of operations on
the database. This has significant performance implications, because I/O
overhead is overwhelmingly the bottleneck by many orders of magnitude in
most programs. [@io]

Standard Library
----------------

The runtime provides native support for an extremely limited subset of
the operations that programmers typically rely on to write programs. The
standard library supplements the functionality of the runtime by
exposing a rich Scala DSL complete with static types, records, math,
collections, and control flow.

### Typing

The runtime natively supports just four dynamic types: flag, real, text,
and null. Dynamic versus static typing is a religious debate among
programmers. Advocates of dynamic typing often mistakenly believe that
type inference and coercive subtyping cannot be provided by a static
type system. In fact, they can. Because static type systems are able to
detect type inaccuracies at compile-time, they allow programmers to
write more concise and correct code. [@typing] The standard library
provides rich static types and features aggressive type inference and
subtype polymorphism.

The standard library supports four **primitive** types. In ascending
order of precedence, they are **boolean**, **int**, **double**, and
**string**. A represents a value of primitive type. There are two kinds
of values. A **constant** is an immutable value, and a **variable** is a
mutable value. Variables may be stored locally in memory or remotely in
a database.

``` {style="Scala"}
    // Creates an integer local variable named x.
    val x = Variable.Local[Int]("x")
    // Creates a floating point remote variable named y.
    val y = Variable.Remote[Double]("y")
    // Assigns y to the sum of x and y.
    y := x + y
    // Assigns x to the product of x and 4.
    x *= 4
    // Does not compile, because y is not an integer.
    x := y
    // Does compile, because floor(y) is an integer.
    x := floor(y)
  
```

### Records

In addition to these primitive types, the standard library also supports
**references** to user-defined types. References use
[Shapeless](https://github.com/milessabin/shapeless) to materialize
compiler macros that permit the fields of an object to be statically
manipulated and iterated. A current limitation is that objects cannot be
self-referential; an object cannot have a field of its own type.

``` {style="Scala"}
    // An example type declaration.
    case class Bar(
      a: String,
    )

    case class Foo(
      b: Int,
      c: Reference[Bar],
      d: Bar
    )

    // Constructs a remote reference to a Foo.
    val x = Reference[Foo](Variable.Remote("x"))
    // Returns the value of the field b.
    x.get(@<'b>@)
    // Does not compile, because z is not a field of Foo.
    x.get(@<'z>@)
    // Serializes x to a JSON string.
    x.asJson
    // Deletes all fields of x and all references.
    x.delete(recursive = true)
    // Constructs a local reference to a Foo.
    val y = Reference[Foo](Variable.Local("y"))
    // Copies x to y.
    y := x
  
```

### Math

The runtime natively supports just nine mathematical operations: $add$,
$sub$, $mul$, $div$, $pow$, $log$, $floor$, $sin$, and $cos$. However,
these primitive operations are sufficient to derive the entire Scala
[math](https://www.scala-lang.org/api/2.12.1/scala/math/index.html)
library using various mathematical identities and approximations. The
$div$, $log$, $sin$, and $cos$ functions can actually be implemented in
terms of the other primitive operations; however, native support for
them was included in the runtime to improve performance. The standard
library provides implementations for all functions enumerated in
Table [\[table:math\]](#table:math){reference-type="ref"
reference="table:math"}.

``` {style="Scala"}
    // Returns the Taylor approximation of inverse sine.
    asin(Pi / 2)
    // Defines the sigmoid function.
    def sigmoid(x: Value[Double]) = exp(x) / (exp(x) + 1)
  
```

### Collections

The runtime has no native support for collections of key-value pairs.
The standard library provides implementations of three fundamental data
structures: **list**, **set**, and **map**. These collections are
mutable and statically-typed. Collections take care of the messy details
of mapping structured data onto a flat namespace, and feature prefetched
iteration. A current limitation is that collections may only contain
primitive types.

``` {style="Scala"}
    // Constructs a map from string to boolean.
    val x = Map[String, Boolean](Variable.Remote("y"))
    // Puts an entry in the map.
    x += "foo" -> true
    // Serializes x to a JSON string.
    x.asJson
    // Constructs a list of integers.
    val x = List[Int](Variable.Local("x"))
    // Increments each element in the list.
    x.foreach(_ + 1)
  
```

### Control Flow

The runtime natively supports control flow operations like $branch$,
$cons$, and $repeat$. However, these operations are syntactically
challenging to express. The standard library uses structural types to
provide support for **if**, **while**, **return**, **assert**, and
**rollback**. The standard library uses an implicitly available parsing
**context** to track modifications made to variables, references, and
collections and to detect when any control flow operations are called.

``` {style="Scala"}
    // If statements.
    If (x < 3) {
      x += 3
    }

    // If/Else statements
    If (y < x) {
      y += 1
    } Else {
      x += 1
    }

    // Ternary operator.
    val z = If (y === x) { y - 1 } Else { y + 1 }

    // While loops.
    While (x <> y) {
      x *= 2
    }
  
```

Compiler
--------

The standard library provides additional functionality that is absent in
the runtime to make it easier to construct programs. However, the
standard library does not address the syntactic challenges of expressing
programs. At times the standard library is forced to use unintuitive
operators like $:=$ and $<>$ and verbose declarations like
$Variable.Remote(``x")$, because of the limitations of implementing a
language within a language.

The compiler translates programs written in the statically-typed,
object-oriented Caustic programming language into operations on the
standard library. For example, consider the following example of a
distributed counter written in Caustic. This program may be run without
modification on any underlying volume and distributed arbitrarily
without error. The Akka project also provides a similar
[implementation](https://git.io/vxS6u) of a backend-agnostic distributed
counter. Their implementation is almost seven times longer.

``` {style="Caustic"}
  module caustic.example

  /**
   * A count.
   *
   * @param value Current value.
   */
  record Total {
    value: Int
  }

  /**
   * A distributed counter.
   */
  service Counter {

    /**
     * Increments the total and returns its current value.
     *
     * @param x Reference to total.
     * @return Current value.
     */
    def increment(x: Total&): Int = {
      if (x.value) x.value += 1 else x.value = 1
      x.value
    }

  }
```

### Implementation

The compiler uses ANTLR [@antlr] to generate a predicated LL(\*) parser
from an ANTLR grammar file. It applies this parser to source files and
walks the resulting parse-tree to generate code compatible with the
standard library. Most statements in Caustic have direct equivalents in
the standard library, and, for the most part, the compiler acts as a
kind of intelligent find-and-replace. However, there are certain aspects
of the compiler that are non-trivial.

First, the compiler performs type inference. It is able statically
verify types and method signatures by maintaining a lexically-scoped
**universe** of the various variables, records, and functions that have
been defined. Because the type system is relatively simplistic, static
types almost never need to be specified. The combination of a static
type system and aggressive type inference allows programs to be both
type-safe and concise.

Second, the compiler is directly integrated in the
[Pants](https://www.pantsbuild.org/) build system. Pants is an
open-source, cross-language build system. Integration into Pants means
that Caustic programs are interoperable with a variety of existing
languages and tooling.

Third, the compiler provides a [TextMate](https://macromates.com/)
bundle that implements syntax highlighting and code completion for most
text editors and IDEs to make it easier for programmers to use the
language.

Evaluation
----------

In this section, we will evaluate the performance of the runtime and
programmability of the programming language. Performance benchmarks were
conducted on a t2.xlarge instance with 32GB of RAM and 8 CPUs.

### Performance

\centering
 

In
Figure [\[figure:runtime-performance\]](#figure:runtime-performance){reference-type="ref"
reference="figure:runtime-performance"}, we first graph execution
latency as programs grow in size. We see that each additional expression
in a program has a constant overhead. In
Figure [\[figure:runtime-performance\]](#figure:runtime-performance){reference-type="ref"
reference="figure:runtime-performance"}, we then graph throughput under
various workloads. We generate programs that read and write the
specified percentage of the key-space uniformly at random and attempt to
simultaneously execute them using the specified number of threads. Note
that the greater the percentage of writes, the greater the likelihood
that a thread will write a key that is simultaneously read or written by
another. Because only one thread can succeed in such a situation, this
causes a reduction in throughput. We see that in read-only workloads,
throughput scales linearly with the number of threads. As we increase
the percentage of writes, throughput falls considerably but still scales
linearly with the number of threads.

### Programmability

Programmability is impossible to measure, because there is no metric by
which two languages can be objectively compared. We will present
implementations of various algorithms in Caustic and in other languages,
but, instead of comparing them with an imprecise measure of
programmability like lines of code or cyclomatic complexity, we will
reserve all judgement and leave it to the reader to verify for
themselves that Caustic is indeed easier to use.

Conclusion
----------

Concurrency is difficult, but it does not need to be. Caustic allows
programmers to build concurrent systems as if they were they were not.
It provides a highly programmable alternative to traditional approaches
for dealing with race conditions and features concise syntax, static
typing, aggressive type inference, and a performant runtime.

We have seen that it is possible to build a robust transactional
programming language over any underlying storage volume that supports
just two operations. In Beaker, we will show that these operations can
be efficiently implemented by a distributed database.

Beaker: A Distributed, Transactional Database
=============================================

In this chapter, we will discuss Beaker. Beaker is a distributed,
fault-tolerant database that efficiently implements the interface on
which transactional programming languages can be built. We will first
motivate the design of Beaker by discussing consensus and its
application to distributed transactions. We will then introduce Beaker
and discuss its implementation. We will conclude with an evaluation of
its performance.

Consensus
---------

The only certainty in distributed systems is that machines will fail.
The key challenge in building fault-tolerant distributed systems is
constructing reliable systems from unreliable components.
 [@reliability] Distributed databases replicate data across a cluster of
machines. By storing copies in different places, distributed databases
are tolerant of individual machine failures.

Replication solves the problem of fault-tolerance, but creates another;
the various replicas must be kept consistent with each other. A naïve
approach is to require all replicas to first agree to apply any
modifications made to the database. This would ensure that all replicas
remained identical. However, this approach is not fault-tolerant. If any
replica were to fail, then no modifications could ever be made to the
database until it recovered. Instead, fault-tolerant distributed
databases require only that a **quorum**, or simple majority, of
replicas to reach agreement before modifications can be safely applied.
This ensures that any quorum of replicas will contain at least one that
has the latest copy of the database while remaining tolerant of a
minority of individual machine failures. Reaching quorum agreement in a
distributed system is known as consensus, and it is a relatively
well-studied problem. In this section, we explore different consensus
algorithms culminating with the approach taken in Beaker.

### Terminology

We define a **proposal** as an abstract operation on a group of replicas
and a **proposer** as the replica that initiates consensus on a
proposal. Over the course of this discussion, we will gradually refine
this definition of a proposal until we arrive at the one used in Beaker.

Replicas communicate by sending messages. We assume that delivered
messages cannot be reordered; if message $A$ was delivered before
message $B$, then $A$ was sent before $B$. In practice, this assumption
is satisfied by most networking protocols including TCP.

### Two-Phase Commit

In Two-Phase Commit, the proposer **prepares** a proposal by first
acquring locks on a quorum of replicas. If it successfully acquires all
locks, the proposer informs all replicas to **learn** the proposal. When
a proposal is learned, its operation is applied and all locks are
subsequently released.

Two-Phase Commit is not fault-tolerant. If the proposer were to fail
after it successfully prepared a proposal but before it requested that
it be learned, the locks that it acquired would never be released and no
new proposals could ever be learned. We will see in Paxos how we can
modify the protocol to guarantee fault-tolerance.

### Paxos

Paxos makes two modifications to Two-Phase Commit to address its
fault-intolerance. First, it associates each proposal with a
monotonically-increasing, globally-unique **ballot** number. Proposals
are totally-ordered and uniquely-identified by their **ballot**. Each
replica keeps track of the latest ballot that it has seen. Second, it
introduces an intermediate **accept** phase to the protocol. We will see
that this additional phase allows the system to recover from proposer
failure. [@paxos]

In Paxos, the proposer prepares a proposal by assigning it a ballot
greater than any it has seen and sending its ballot to a quorum of
replicas. If the ballot is greater than any it has seen, a replica
**promises** not to accept any proposal less than it and returns any
proposal that it has already accepted. Otherwise, the replica ignores
the request. Intuitively, ballots function as a kind of implicit lock
that the proposer holds until another proposer prepares a greater
ballot. If a majority of replicas do not respond to its prepare request,
the proposer retries with a greater ballot. Otherwise, the proposer
selects a proposal to be accepted. If any replica returned an accepted
proposal, then the proposer must select the latest accepted proposal and
set its ballot to the one that it prepared. Otherwise, the proposer
selects its own proposal. Intuitively, this allows the system to recover
when a proposer fails after convinced a majority to accept its proposal
but before it could be learned. A replica accepts a proposal if and only
if it has not promised not to. When a replica accepts a proposal, it
requests that all replicas learn it. A replica learns a proposal when a
majority of replicas have requested that it be learned. When a replica
is learned, its operation is applied and any accepted proposals are
removed. Intuitively, this releases the implicit lock held by the
proposer and consensus to begin on another proposal.

Paxos guarantees that all non-faulty replicas will learn proposals in
the same order. Often, this guarantee is unnecessary because a large
number of operations on a distributed system are commutative, so they
may be performed in any order. For example, reads and writes to
different keys in a database may be performed in any order without
compromising consistency. We will see in Generalized Paxos that we can
exploit commutativity to improve performance.

It is known that no deterministic fault-tolerant consensus protocol can
guarantee progress in an asynchronous network. [@consensus] Paxos is no
exception. If a higher ballot is continuously prepared before any
proposal can be accepted, no proposal will ever be learned.
Implementations of Paxos typically elect a distinguished replica, called
a **leader**, to which all other replicas forward their proposals to
guarantee liveness. Whenever leaders fail, replicas run an instance of
Paxos to acquire leadership of the cluster. The reliance on the
existence of a single, stable leader is both a important simplifying
assumption and a performance limitation. If there exists a leader, then
prepare messages are superfluous. Intuitively, the leader implicitly
holds a lock on all replicas because no other replica can initiate
proposals. This allows proposals to be learned in just two message
delays. However, the reliance on the leader to initiate all proposals is
also a significant bottleneck at scale. The entire system moves at the
rate of the leader. In fact, this is the fundamental limitation in
implementations of Paxos like ZooKeeper  [@zookeeper] and
Chubby [@chubby]. We will see in Egalitarian Paxos that we can remove
the dependence on a leader to improve performance.

### Generalized Paxos

Generalized Paxos [@generalized-paxos] addresses the scalability of
Paxos by exploiting commutativity. An operation $A$ commutes with $B$ if
performing $A$ after $B$ has the same effect as performing $B$ after
$A$. For example, addition is commutative but division is not;
$4 + 3 = 3 + 4$ but $\frac{4}{3} \ne \frac{3}{4}$. In fact, most
operations on a distributed database are commutative. Reads commute with
each other and reads and writes to different keys commute.

Generalized Paxos associates each proposal with a sequence of
operations. We say that proposal $A$ is **equivalent** to proposal $B$
if all non-commutative operations in $A$ and $B$ are in the same order.
All equivalent proposals have the same effect. Generalized Paxos permits
replicas to learn different proposals as long as they are equivalent.

In Generalized Paxos, proposers do not forward their requests to leader.
Instead, they immediately request that all replicas accept their
proposed operation. A replica appends the operation to their currently
accepted proposal and requests that all replicas learn it. A proposal is
learned when a majority of replicas have requested that it or an
equivalent proposal be learned. If no majority of replicas can agree on
the ordering of non-commutative operations, it is the responsibility of
the leader to select one and to run an instance of Paxos to convince the
other replicas to accept its choice before resuming normal operation.

Like Paxos, Generalized Paxos relies on the existence of a single,
stable leader to mediate ordering disagreements between replicas and
guarantees that all commutative operations will be learned in two
message delays. Unlike Paxos, it does not require all proposals to
originate from the leader. If most operations are commutative, the
leader will rarely be required to arbitrate. However, the existence of a
leader can still be a scalability bottleneck. We will see in Egalitarian
Paxos that we can remove the dependence on a leader to improve the
performance of the system.

### Egalitarian Paxos

Egalitarian Paxos [@epaxos] makes a subtle modification to Generalized
Paxos to remove its dependence on a leader. Egalitarian Paxos associates
with proposal with a directed acyclic graph of operations in which each
edge corresponds to a dependency between two operations. The benefit of
using a directed acyclic graph is that its various strongly connected
components can be performed in parallel without impacting consistency.
This has huge ramifications for performance, particularly in databases
because reads and writes are relatively expensive operations.

In Egalitarian Paxos, an operation depends on all accepted proposals for
which it does not commute. The proposer builds a dependency graph for a
proposal from any proposals that it has already accepted and requests
that all replicas accept it. A replica supplements the dependency graph
of the proposal with any proposals that it has accepted and requests
that the result be learned. If no majority of replicas can agree on the
dependency graph of a proposal, it is the responsibility of the proposer
to select one and to run an instance of Paxos to convince the other
replicas to accept its choice before resuming normal operation.

Egalitarian Paxos implicitly assumes that operations are idempotent,
because an operation may be in the dependency graph of multiple
proposals. An operation $A$ is idempotent if repeated non-sequential
applications of $A$ have the same effect as a single application of $A$.
For example, multiplication by one is idempotent but by two is not;
$4 * 1 = 4 * 1 * 1$ but $4 * 2 \ne 4 * 2 * 2$. This assumption will
become important when we use Egalitarian Paxos to implement distributed
transactions.

Distributed Transactions
------------------------

We now use Egalitarian Paxos to implement distributed transactions. We
begin by concretely defining the underlying system. We then describe the
consensus protocol and conclude with a rough proof of correctness.

### Terminology

A **database** is a key-value store that supports two operations:
**read** and **write**. Databases guarantee durability; if a key is
written, then subsequent reads will always return the updated value.
Like Caustic, we will associate each key with a **revision**, or
versioned value, with the same conflict relationship as before. Reads
and writes on the database return or update the revisions of a set of
keys. Operations on a database are non-transactional and can fail
arbitrarily.

We define our abstract operation as a **transaction**. A transaction is
composed of a set of dependent versions, called its **readset**, and a
set of updates, called its **writeset**. We say that two transactions
**conflict** if there exists a key in the readset or writeset of either
transaction that is in the writeset of the other. A transaction may be
**committed** on a database by applying the updates in its writeset if
and only it depends on the latest version of every key in its readset.
We say that a transaction is **committable** if it can be committed. In
order to guarantee idempotency of commit, any key in the writeset of a
transaction must also be present in its readset.

A **cache** is a write-through database. Because dependencies are
validated on commit, keys may be speculatively read from cache without
sacrificing consistency. Beaker supports multi-level cache hierarchies
over the underlying database. Revisions may be **fetched** or
**updated** in cache. Cache coherency is maintained by updating the
cache whenever transactions are committed on the underlying database.

An **archive** is a transactional database. Archives use an **executor**
to linearize conflicting transactions on the underlying database. An
executor schedules transactions in **epochs**. Transactions are
scheduled in the earliest epoch such that they do not conflict with any
transaction in a later epoch. This guarantees consistency and isolation;
transactions see the effect of any previously committed conflicting
transaction and conflicting transactions are never committed
simultaneously.

We define our replica as a **beaker**. Beakers use a variation of
Egalitarian Paxos to coordinate distributed transactions with several
desirable properties. First, non-conflicting transactions may be
simultaneously committed. Second, databases with stale revisions are
automatically repaired. Third, transactions may be committed as long as
at least a majority of beakers are operational.

### Protocol

Beaker makes two key modifications to Egalitarian Paxos. First, it
associates each proposal with a set of non-conflicting transactions,
called its **commits**, and a set of revisions, called its **repairs**.
We say that a proposal $A$ **conflicts** with a proposal $B$ if they
have any conflicting commits. We say that a proposal $A$ is
**equivalent** to proposal $B$ if their commits are the same. A proposal
$A$ may be **merged** with a proposal $B$ by taking the maximum of their
ballots, combining their commits choosing the transaction in the newer
proposal in the case of conflicts, and combining their repairs choosing
the highest revision in the case of duplicates. Second, it introduces an
intermediate **get** phase to the protocol. We will see that this
additional phase allows the system to repair databases with stale
revisions.

In Beaker, the proposer prepares a proposal by sending it to all
beakers. If a beaker has not made a promise to a newer proposal, it
responds with a promise not to accept any proposal that conflicts with
the proposal it returns. If it has already accepted any conflicting
proposals, it merges them together and returns the result. Otherwise, it
returns the prepared proposal with a zero ballot. If the proposer does
not receive a majority of promises, it retries with a higher ballot.
Otherwise, it merges the returned promises together into a single
proposal. If the merged proposal is not equivalent to its prepared
proposal, the proposer retries with a higher ballot. Otherwise, the
proposer gets the revisions of all keys that are read by the proposal
from a majority of beakers. The proposer discards all transactions that
cannot be committed given the latest returned revisions and sets the
repairs of the proposal to the latest revisions of keys that are read -
but not written - by the proposal for which any two beakers have
different revisions. The proposer then requests all beakers to accept
the proposal. A beaker accepts a proposal if it has not promised not to.
If a beaker accepts a proposal, it requests that all other beakers learn
the proposal. A beaker learns a proposal when a majority of beakers have
requested that it be learned. When a beaker learns a proposal, it
commits its transactions and repairs on its archive.

### Correctness

We will make the assumption of **connectivity**; all learn messages are
always delivered. This ensures that the proposer will always learn that
its proposal was learned. This is a necessary requirement because it
allows clients to know with certainty whether or not their transaction
was committed. In practice, this assumption can be weakened by adding an
additional decide phase to the protocol. Whenever a proposal is learned
by a beaker, it informs all other beakers that the proposal was decided.
As long as at least one beaker receives a majority of learn messages and
at least one decide message is delivered to each beaker, any proposer is
guaranteed to learn that its proposal was learned. A proposal is learned
by a beaker when it receives either a majority of learn requests or a
decide message. The decide phase introduces additional partition
tolerance and comes at no additional cost if the network is perfectly
reliable. It does, however, increase the number of messages sent between
beakers which could saturate the network. The assumption of connectivity
is reasonable if beakers are co-located in the same data center, but is
unreasonable if they are geographically separated.

We will also make the assumption of **reliability**; at least a majority
of replicas will atomically apply each learned proposal. This ensures
that any majority of replicas will contain at least one that has the
latest revision of each key-value pair. Our assumption of reliability is
substantially weaker than the one implicitly made by the previously
presented Paxos algorithms; they assume that a majority of replicas
atomically apply *every* learned proposal. We will see that consistency
is guaranteed even if no replica has applied every learned proposal.

We will also make use of the fact of **quorum intersection**; any two
majorities of beakers must have at least one in common. Verification of
this property is trivial and is left to the reader.\

Any proposal $A$ that is accepted by a majority will eventually be
learned by everyone.

By quorum intersection, at least one promise will contain $A$ until $A$
has been learned by sufficiently many beakers that it no longer has been
accepted by a majority. But if $A$ was learned by any beaker, at least a
majority of beakers must have requested that it be learned. By
connectivity, if a majority of beakers requested that $A$ be learned
then all beakers will learn $A$. Therefore, $A$ will eventually be
learned by everyone.

Note that we guarantee only that a proposal will eventually be learned
once it has been accepted by a majority, but we do not guarantee that a
proposal will ever accepted by a majority. It is possible for proposers
to continuously prepare higher ballot proposals before any can be
accepted by a majority. In practice, by retrying with exponentially
jittered backoffs the likelihood of such an event asymptotically
approaches zero.\

If proposal $A$ is accepted by a majority, then any conflicting proposal
$B$ that is subsequently prepared will always be learned after $A$.

Because $A$ was accepted by a majority before $B$, the majority that
accepted $A$ before $B$ will request that $A$ be learned before $B$.
Because messages are always delivered in order and by connectivity, $B$
will always be learned after $A$.

Let $R$ denote the repairs for a proposal $A$ that has been accepted by
a majority. Any proposal $B$ that is accepted by majority and conflicts
with $A + R$ but not $A$ commutes with $A + R$.

Because $B$ conflicts with $A + R$ but not $A$, $B$ must read a key $k$
that is read by $A$. By reliability, $B$ must read the latest version of
$k$. Suppose that $B$ is committed first. Because $B$ reads and does not
write $k$, $A + R$ can still be committed. Suppose that $A + R$ is
committed first. Because $A + R$ writes the latest version of $k$ and
$B$ reads the latest version, $B$ can still be committed. Therefore, $B$
commutes with $A + R$.

If a proposal $A$ is accepted by a majority, its transactions are
committable.

Suppose there exists a transaction that cannot be committed. Then, the
transaction must read a key for which there exists a newer version. This
implies that there exists a proposal $B$ that was accepted by a majority
after, but learned before $A$ that changes a key $k$ that is read by
$A$. By serializability, $B$ cannot conflict with $A$. Therefore, $B$
must repair $k$. By commutativity of repairs, $A$ may still be
committed.

### Reconfiguration

In practical systems, beakers may join or leave the cluster arbitrarily
as the cluster grows or shrinks in size. In this section, we describe
how beakers are bootstrapped when they join an existing cluster. We say
that a beaker is **fresh** when it initially joins a cluster. In order
to guarantee correctness, fresh beakers must be immediately populated
with the latest revision of every key-value pair. Otherwise, if $N + 1$
fresh beakers join a cluster of size $N$ it is possible for a majority
of beakers to consist entirely of fresh beakers. This violates the
assumption of reliability.

A naive solution might be for the fresh beaker to propose a read-only
transaction that depends on the initial revision of every key-value pair
in the database and conflicts with every other proposal. Then, the fresh
beaker would automatically repair itself in the process of committing
this transaction. However, this is infeasible in practical systems
because databases may contain arbitrarily many key-value pairs. This
approach would inevitably saturate the network. Furthermore, it prevents
any proposals from being accepted in the interim.

We can improve this solution by decoupling bootstrapping and consensus.
A fresh beaker joins the cluster as a partial member, called a
**learner**, that learns proposals, but is not involved in consensus.
The fresh beaker **scans** a majority of full members, called
**acceptors**, and repairs its own archive with the latest returned
values. By assumption of reliability, it is guaranteed to have the
latest value of every key. It then joins the cluster as an acceptor.
This approach consumes less bandwidth and permits concurrent proposals.

Each beaker maintains its own view of the cluster configuration which it
sends alongside any proposal that it proposes. If the view is outdated,
beakers inform the proposer of the new configuration. If a beaker has
already accepted a proposal from an outdated view, that proposal must be
completed before the cluster can move to a new configuration. This
ensures that reconfiguration is consistent across the cluster.

Evaluation
----------

### Contention

In this section, we examine the performance of Beaker under contention
both from a theoretical and a practical perspective. We begin with a
mathematical treatment of consensus and conclude with an empirical
verification of our analysis.

We may model the number of retries required to successfully commit a
transactions as a negative binomial distribution. A negative binomial
distribution $NB(p, r)$ measures the number of failed events each
occurring with probability $p$ before $r$ successes are encountered. In
this case, $p$ represents the **contention probability**, or the
likelihood that any two concurrent transactions conflict. Therefore, the
distribution of total attempts required to successfully commit any
transaction as $A \sim 1 + NB(p, 1)$. We may then use known results
about the negative binomial distribution to make predictions about the
how the system will behave under contention.

We must first make concrete our definition of contention probability.
Consider two sets $X$ and $Y$ of size $l$ selected uniformly at random
from the set \[0, n). The contention probability is the likelihood that
$X$ and $Y$ are not disjoint. Intuitively, the probability that $X$ and
$Y$ are disjoint is equal to the likelihood that $Y$ is drawn from the
set $[0, n) \setminus X$. Therefore, the probability that $X$ and $Y$
are not disjoint is equal to $1 - \frac{C(n - 1, l)}{C(n, l)}$ where
$C(a, b)$ is the number of ways to choose $b$ items from a set of size
$a$. We may extend this notion of uniformly random sets to uniformly
random transactions. Suppose two transactions $T_1$ and $T_2$ each write
$l$ keys chosen uniformly at random from a key space of size $n$. The
probability that they will conflict is exactly equal to the probability
that $X$ and $Y$ are not disjoint.

To verify that this theoretical definition of contention probability
matches empirical results I conducted a Monte Carlo simulation in which
two threads repeatedly generate and commit $M$ transactions that
increment a uniformly random set of $l$ keys and monitor the total
number of failures. If the hypothesis about the negative binomial
distribution is correct, transactions should conflict on average $p * M$
times. I found this to be empirically validated, but the reader is
encouraged to verify this claim for themselves.

We assume for the sake of simplicity that there exist exactly two
proposers that are simultaneously committing transactions. In practice,
most systems will have many more active proposers. Unfortunately, I lack
the mathematical acumen to analytically derive contention probability in
the general case. However, the contention probability may be numerically
approximated in the general case by changing the number of threads in
the Monte Carlo simulation.

We may use our empirically verified model of contention to make
predictions about the performance degradation of the system under
varying degrees of contention. Given a key-space of size $n$ and an
average transaction of size $l$, we would expect each transaction to
require $\mathbb{E}[A] = 1 + \frac{p}{1 - p}$ attempts and for the
average throughput of the system to decrease by the same amount.

Conclusion
----------

\appendix
Caustic Addemdum
================

\centering
  Expression        Description
  ----------------- -----------------------------------------------------------
  add(x, y)         Sum of $x$ and $y$.
  both(x, y)        Bitwise AND of $x$ and $y$.
  branch(c, p, f)   Executes $p$ if $c$ is true, or $f$ otherwise.
  cons(a, b)        Executes $a$ and then $b$.
  contains(x, y)    Returns whether or not $x$ contains $y$.
  cos(x)            Cosine of $x$.
  div(x, y)         Quotient of $x$ and $y$.
  either(x, y)      Bitwise OR of $x$ and $y$.
  equal(x, y)       Returns whether $x$ and $y$ are equal.
  floor(x)          Floor of $x$.
  indexOf(x, y)     Returns the index of the first occurrence of $y$ in $x$.
  length(x)         Returns the number of characters in $x$.
  less(x, y)        Returns whether $x$ is strictly less than $y$.
  load(n)           Loads the value of the variable $n$.
  log(x)            Natural log of $x$.
  matches(x, y)     Returns whether or not $x$ matches the regex pattern $y$.
  mod(x, y)         Remainder of $x$ divided by $y$.
  mul(x, y)         Product of $x$ and $y$.
  negate(x)         Bitwise negation of $x$.
  pow(x, y)         Returns $x$ raised to the power $y$.
  prefetch(k, s)    Reads keys at $k/i$ for $i$ in $[0, s)$.
  read(k)           Reads the value of the key $k$.
  repeat(c, b)      Repeatedly executes $b$ while $c$ is true.
  rollback(r)       Discards all writes and returns $r$.
  sin(x)            Sine of $x$.
  slice(x, l, h)    Returns the substring of $x$ between $[l, h)$.
  store(n, v)       Stores the value $v$ for the variable $n$.
  sub(x, y)         Difference of $x$ and $y$.
  write(k, v)       Writes the value $v$ for the key $k$.

  : Instruction Set

[\[table:expressions\]]{#table:expressions label="table:expressions"}

\centering
  Function      Description
  ------------- --------------------------------------------------------------
  abs(x)        Absolute value of $x$.
  acos(x)       Cosine inverse of $x$.
  acot(x)       Cotangent inverse of $x$.
  acsc(x)       Consecant inverse of $x$.
  asec(x)       Secant inverse of $x$.
  asin(x)       Sine inverse of $x$.
  atan(x)       Tangent inverse of $x$.
  cbrt(x)       Cube root of $x$.
  ceil(x)       Smallest integer greater than or equal to $x$.
  cos(x)        Cosine of $x$.
  cosh(x)       Hyperbolic cosine of $x$.
  cot(x)        Cotangent of $x$.
  coth(x)       Hyperbolic cotangent of $x$.
  csc(x)        Cosecant of $x$.
  csch(x)       Hyperbolic cosecant of $x$.
  exp(x)        Exponential of $x$.
  expm1(x)      Equivalent to $e^x - 1$.
  floor(x)      Largest integer less than or equal to $x$.
  hypot(x, y)   Hypotenuse of the triangle with base $x$ and height $y$.
  log(x)        Natural logarithm of $x$.
  log10(x)      Log base 10 of $x$.
  log1p(x)      Equivalent to $\log (x + 1)$.
  pow(x, y)     Power of $x$ to the $y$.
  random()      Uniformly random number on \[0, 1).
  rint(x)       Closest integer to $x$, rounding to the nearest even number.
  round(x)      Closest integer to $x$, rounding up.
  round(x, y)   Closest multiple of $y$ to $x$.
  sec(x)        Secant of $x$.
  sech(x)       Hyperbolic secant of $x$.
  signum(x)     Sign of $x$.
  sin(x)        Sine of $x$.
  sinh(x)       Hyperbolic sine of $x$.
  sqrt(x)       Square root of $x$.
  tan(x)        Tangent of $x$.
  tanh(x)       Hyperbolic tangent of $x$.

  : Math

[\[table:math\]]{#table:math label="table:math"}

Beaker Addemdum
===============

\printbibliography
