# Chapter 2: Thread Safety

Writing thread-safe code is, at its core, about managing access to _state_, and in particular to _shared,
mutable state_.

By _shared_, we mean that a variable could be accessed by multiple threads; by _mutable_, we mean that its value could
change during its lifetime. We may talk about thread safety as if it were about _code_, but what we are really trying
to do is protect _data_ from uncontrolled concurrent access.

Whenever more than one thread accesses a given state variable, and one of them might write to it, they all must
coordinate their access to it using synchronization.

If multiple threads access the same mutable state variable without appropriate synchronization, _your program is
broken_. There are three ways to fix it:

* _Don’t share_ the state variable across threads;
* Make the state variable _immutable_; or
* Use _synchronization_ whenever accessing the state variable.

It is far easier to design a class to be thread-safe than to retrofit it for thread safety later.

When designing thread-safe classes, good object-oriented techniques—encapsulation, immutability, and clear
specification of invariants—are your best friends.

## 2.1 What is thread safety?

A class is _thread-safe_ if it behaves correctly when accessed from multiple threads, regardless of the scheduling
or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization
or other coordination on the part of the calling code.

No set of operations performed sequentially or concurrently on instances of a thread-safe class can cause an instance
to be in an invalid state.

Thread-safe classes encapsulate any needed synchronization so that clients need not provide their own.

### 2.1.1 Example: a stateless servlet

Stateless objects are always thread-safe.

## 2.2 Atomicity

The possibility of incorrect results in the presence of unlucky timing is so important in concurrent programming that
it has a name: a _race condition_.

<details><summary>race condition or data race? </summary>The term _race condition_ is often confused with the related
term _data race_, which arises when synchronization is not used to coordinate all access to a shared non-final field.
You risk a data race whenever a thread writes a variable that might next be read by another thread or reads a variable
that might have last been written by another thread if both threads do not use synchronization; code with data races
has no useful defined semantics under the Java Memory Model. Not all race conditions are data races, and not all data
races are race conditions, but they both can cause concurrent programs to fail in unpredictable ways.</details>

### 2.2.1 Race conditions

The most common type of race condition is _check-then-act_, where a potentially stale observation is used to make a
decision on what to do next.

Using a potentially stale observation to make a decision or perform a computation is what characterizes most race
conditions. This type of race condition is called _check-then-act_: you observe something to be true (file X doesn't
exist) and then take action based on that observation (create X); but in fact the observation could have become
invalid between the time you observed it and the time you acted on it (someone else created X in the meantime),
causing a problem (unexpected exception, overwritten data, file corruption).

### 2.2.2 Example: race conditions in lazy initialization

A common idiom that uses check-then-act is _lazy initialization_.

The following snippet has race conditions (**don't do this!**) that can undermine its correctness. Say that threads
**A** and **B** execute `getInstance` at the same time. **A** sees that instance is `null` , and instantiates a new
`ExpensiveObject`. **B** also checks if instance is `null` . Whether instance is `null` at this point depends
unpredictably on timing, including the vagaries of scheduling and how long **A** takes to instantiate the
`ExpensiveObject` and set the instance field. If instance is `null` when **B** examines it, the two callers to
`getInstance` may receive two different results, even though `getInstance` is always supposed to return
the same instance.

```java
@NotThreadSafe
public class LazyInitRace {
  private ExpensiveObject instance = null;
  
  public ExpensiveObject getInstance() {
    if (instance == null) {
      instance = new ExpensiveObject();
    }
    
    return instance;
  }
}
```

### 2.2.3 Compound actions

Operations A and B are _atomic_ with respect to each other if, from the perspective of a thread executing A,
when another thread executes B, either all of B has executed or none of it has. An _atomic_ operation is one that
is atomic with respect to all operations, including itself, that operate on the same state.

We refer collectively to check-then-act and read-modify-write sequences as _compound actions_: sequences of
operations that must be executed atomically in order to remain thread-safe.

When a single element of state is added to a stateless class, the resulting class will be thread-safe if the state is
entirely managed by a thread-safe object.

Where practical, use existing thread-safe objects, like `AtomicLong`, to manage your class’s state. It is simpler
to reason about the possible states and state transitions for existing thread-safe objects than it is for arbitrary
state variables, and this makes it easier to maintain and verify thread safety.

## 2.3 Locking

When multiple variables participate in an invariant, they are not _independent_: the value of one constrains the
allowed value(s) of the others. Thus, when updating one, you must update the others _in the same atomic operation_.

To preserve state consistency, update related state variables in a single atomic operation.

### 2.3.1 Intrinsic locks

Java provides a built-in locking mechanism for enforcing atomicity: the _synchronized_ block. Every Java object can
implicitly act as a lock for purposes of synchronization; these built-in locks are called _intrinsic locks_ or
_monitor locks_.

Intrinsic locks in Java act as _mutexes_ (or mutual exclusion locks), which means that at most one thread may own
the lock.

### 2.3.2 Reentrancy

Reentrancy means that locks are acquired on a per-thread rather than per-invocation basis.

Intrinsic locks are _reentrant_, if a thread tries to acquire a lock that it already holds, the request succeeds.

## 2.4 Guarding state with locks

Holding a lock for the _entire duration_ of a compound action can make that compound action atomic.

If synchronization is used to coordinate access to a variable, it is needed _everywhere that variable is accessed_.

When using locks to coordinate access to a variable, the _same_ lock must be used wherever that variable is accessed.

For each mutable state variable that may be accessed by more than one thread, _all_ accesses to that variable must
be performed with the _same_ lock held. In this case, we say that the variable is _guarded by_ that lock.

Every shared, mutable variable should be guarded by exactly one lock. Make it clear to maintainers which lock that is.

For every invariant that involves more than one variable, _all_ the variables involved in that invariant must be
guarded by the _same_ lock.

## 2.5 Liveness and performance

Acquiring and releasing a lock has some overhead, so it is undesirable to break down synchronized blocks _too_ far.

There is frequently a tension between simplicity and performance. When implementing a synchronization policy, resist
the temptation to prematurely sacrifice simplicity (potentially compromising safety) for the sake of performance.

Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network
or console I/O.
