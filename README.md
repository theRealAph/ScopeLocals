# ScopeLocals

## Rationale

The need for scope locals arose originally from Project Loom. Loom
enables a new style of Java programming where Threads are not a scarce
resource to be carefully managed by thread pools but are much more
abundant, limited only by memory. If we are to be able to create large
numbers of threads &mdash; potentially millions &mdash; we'll need to
make sure that all of the per-thread structures scale well.

ThreadLocals, and in particular inheritable thread locals, are a pain
point in the design of Loom. Every time a new Thread instance is
created its set of ThreadLocals (a kind of hash map) is cloned. This
is necessary because a Thread's set of ThreadLocals is, by design,
mutable, so it cannot be shared. For that scalability reason, Loom's
lightweight "Virtual Threads" do not support inheritable ThreadLocals.
However, inheritable ThreadLocals have a useful role in conveying
context information from parent thread to child, so we need something
else which will fill the gap.

Ideally we'd like to have a zero-copy operation when creating Threads,
so that inheriting context requires only copying a pointer from the
creating Thread (the parent) into the new Thread (the child). The
inherited context should be immutable, so it can be shared between
threads and other contexts without the need to copy it.

## Dynamically-scoped values

The idea is to support something like a "special variable" in Common
Lisp. This is a dynamically-scoped variable which acquires a value on
entry to a lexical scope, and when that scope terminates, its previous
value (or none) is restored. We also intend to support thread
inheritance for scope locals, so that parallel constructs can easily
set a value in the outer scope before threads start.

One useful way to think of ScopeLocals is as invisible, effectively
final, parameters which are passed to every method. These will be
visible within the "dynamic scope" of a ScopeLocal's binding operation
(i.e. the set of methods invoked within the binding scope and
transitively by them.)

## Syntax

```
  // Declare scope locals x and y
  static final ScopeLocal<MyType> x = ScopeLocal.forType(MyType.class);
  static final ScopeLocal<MyType> y = ScopeLocal.forType(MyType.class);

  {
    ScopeLocal.where(x, expr1)
              .where(y, expr2)
              .run(() -> ... code that uses x.get() and y.get() ...);
  }
```

Note that the declarations of ScopeLocals `x` and `y` have an explicit
type parameter. This allows us to make a strong guarantee that if any
attempt is made to bind an object of the wrong type to a ScopeLocal
we'll immediately throw a ClassCastException.

Also, the declarations of the ScopeLocals `x` and `y` have the usual
Java access modifiers. Even though a scope local is passed to every
method in its dynamic scope, a method will only be able to use get()
if that ScopeLocal's name is visible to that method. So, sensitive
security information can be passed through a stack of non-privileged
invocations.

## A simple example

The following example uses a ScopeLocal to make credentials available
to callees.

```
   private static final ScopeLocal<Credentials> CREDENTIALS = ScopeLocal.forType(Credentials.class);

   Credentials creds = ...
   ScopeLocal.where(CREDENTIALS, creds).run(creds, () -> {
       :
       Connection connection = connectDatabase();
       :
   });

   Connection connectDatabase() {
       Credentials credentials = CREDENTIALS.get();
       :
   }
```

## Inheritance

ScopeLocals are either inheritable or non-inheritable. The
ineritability of a ScopeLocal is determined by its declaration,like
so:

```
  static final ScopeLocal<MyType> x = ScopeLocal.inheritableForType(MyType.class);
```

Whenever Thread instances (virtual or not) are created, the set of
currently-bound inheritable ScopeLocals is inherited by the child
thread. In addition, a `Snapshot()` operation which returns the
current set of inheritable ScopeLocals is provided so that other kinds
of tasks (e.g. a `ForkJoinTask`) can inherit ScopeLocals. This is done
by means of a `ScopeLocal.runWithSnapshot` method.

## Compared with ThreadLocals

ScopeLocals are by design less general-purpose than ThreadLocals. The
current value of a ScopeLocal cannot be changed within a method and
neither can its value be returned to a caller. On the upside, this
means ScopeLocals, unlike ThreadLocals can't leak memory if a
programmer sets but "forgets" to remove a value once it's no longer
needed.

About security: because a ScopeLocal is only bound to a value in an
invocation, and when the scope of that invocation terminates the value
of the ScopeLocal its previous value (or none) is restored, we can
make some very strong guarantees. Firstly, bound values can never
"leak" into the context of a caller, whch makes it safe to pass
sensitive security context to a subtask. Also, for the same reason,
within the context of a method we know that the value of a ScopeLocal
is invariant.

Although an API does not have any particular performance properties or
guarantees, the ScopeLocal API was designed with efficiency in mind
and it would be remiss not to talk about what a programmer can
reasonably expect.

### `get()`

`ThreadLocal.get()` has time complexity of O(1) because a hash table
is used. The first time `ScopeLocal.get()` used its time complexity is
O(n), with n the size of the current set of bindings. However, we use
a small per-thread cache so that repeated `get()` of the same
ScopeLocal is, probabilistically, O(1). In practice `get()` is
extremely fast, about the same as a local variable. (C2 even hoists
the result of `ScopeLocal.get()` into a register if a value is used
repeatedly.) Also, it's not necessary for `get()` to perform a
checkcast because we know that when a ScopeLocal is bound its type is
checked.

### Inheritance

ThreadLocal inheritance is O(n) in both space and time. ScopeLocal
inheritance is O(1), almost immeasurably small.

### Binding operations

Here the performance is broadly similar. ThreadLocal.set() is a
HashTable insertion, and ScopeLocal the creation of a binding object
and its insertion into the list of currently-bound
ScopeLocals. There's also the creation of a Callable or a Runnable:
pretty cheap but non-zero.

The first usage of `ThreadLocal.set()` is fairly expensive, but
subsequent usages of `set()` on the same ThreadLocal are relatively
cheap because it's not necessary to create a new object. ThreadLocal
wins here by a small margin, as long as there is no cleanup `remove()`
operation.

### Snapshot

Here there's no comparison: ThreadLocal has no equivalent.

