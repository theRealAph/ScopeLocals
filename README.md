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
thread. In addition, a `Snapshot()` operation is provided so that
other kinds of tasks (e.g. a `ForkJoinTask`) can inherit ScopeLocals.
