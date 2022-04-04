# Extent-local variables

## Summary

Define a standard, lightweight way to pass contextual information from
a caller to callees in the currently executing thread, and in some
cases to a group of threads. In particular, extent locals are intended
to replace `ThreadLocal`s for sharing information with a very large
number of threads. This usage especially applies to virtual threads.

## Goals

- *Ease of use* — Providing a clear programming model to share data
  within and across threads in order to simplify reasoning about
  data flow.
- *Comprehensibility* — Making the lifetime of shared data more
  clearly visible in the program code. 
- *Performance* — Treating shared data as immutable so as to allow
  sharing among large numbers of threads. 

## Non-Goals

It is not a goal to change the Java Programming Language.

It is not a goal for this new programming model to require use
of interface `AutoCloseable`.

It is not a goal to require migration away from thread local
variables.

## Motivation

Programs are usually built out of components that contribute independent,
complementary functionality. For example a networked server may combine
business logic to handle service requests with components such as a database
driver providing data persistence and a logging framework that records
progress or noteworthy events. As a security measure, the thread which
serves an individual request might be provided with a `Permissions` token
which controls access to the operations provided by these server components.

Per-thread context such as the permissions token, would normally be communicated
via method arguments. However, in cases like this
where the data needs to be pushed through calls to the business logic code
it simplifies the implementation if the server can share context data with the
components via some alternative channel. 
    
    Thread 1                               ...                    Thread N
    Logger.log()   get  <---------------------+   +--------> get  Logger.log()
    ...                                       |   |               ...
    ...                                    PERMISSIONS            ...                     
    AppLogic.handleRequest()                  |   |               AppLogic.handleRequest()
    ServerFrameWork.processRequest() :  set --+   +--------- set  ServerFrameWork.processRequest()
     
What makes this complicated is that each thread needs its own independent
permissions so it also needs its own independent channel. 

### Current Capabilities

Developers traditionally turn to `ThreadLocal` in situations like this. It
provides developers with an independent variable for each thread to
share their own copy of data.

    class ServerFramework {
      // 1. Provide a per-thread channel betwen the framework and its components
      final static ThreadLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAdmin() ? LOG|DATABASE : LOG);
        Permissions permissions = new Permissions(p);
        // 2. Set permissions for request thread and run app logic
        PERMISSIONS.set(permissions);
        AppLogic.handleRequest(request, response);
      }
      ...
    }

    class Logger {
      void warn(String message)  throws PermissionException {
        // 3. Get permissions for request thread and check  
        Permissions permissions = ServerFramework.PERMISSIONS.get();
        if (!permissions.allowed(LOG) {
          throw new PermissionException(...)
        }
        // 4. Ok to write the warning
        write(logFile, "WARNING : %s %s".format(timeStamp(), message));
      }
      ...
    }

    class DBDriver {
      DBConnection open(...) throws PermissionException {
        // 5. Get permissions for request thread and check
        Permissions permissions = ServerFramework.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) {
          throw new  PermissionException(...)
        }
        // 6. Ok to connect to a database
        return DBPool.newConnection(...);
      }
      ...
    }


The `ThreadLocal` declared at 1. in static field `PERMISSIONS` provides a
way to ensure that each thread handling a request can have its own independent
permissions. A `ThreadLocal` serves as a key that is used to look up
a `PermissionImpl` value for the current thread. So, there is _not_ exactly one
incarnation of the field shared across all instances of Foo; instead, there are
_multiple_ incarnations of the field, one per thread, and the incarnation of
`PERMISSIONS` that is used when code performs sets or gets the `ThreadLocal`
depends on the thread which is executing the code.

Using a `ThreadLocal` avoids the need to pass a `Permissions` object as
an argument to calls from the service handler through the business logic code
and into the database driver or logger code. The declaration at 1. include an
initialization that assigns a reference to a `ThreadLocal` object to field
`PERMISSIONS`. Each server thread that handles an incoming request still needs
to call the `set()` method at 2. This ensures that the incarnation of field
`PERMISSIONS` specific to the handler thread identifies a set of permissions
appropriate to the type of request. The server is now ready to execute the
business logic.

The logger and database components call the `get()` method at 3. and 5. to
retrieve the permissions for the current thread. They ensure that the
operation requested by the business logic is permitted before proceeding
to execute it at 5. and 6. 

This example demonstrates that `ThreadLocal`s can be used to implement the
behaviour needed for this case. However, there are problems with their design
that affect even this sort of well structured use.
 
- *Unconstrained Mutability*  —  Every thread-local variable is _mutable_:
  that is to say, any code that can access a `ThreadLocal`'s `get()` can
  also `set()` it to a different value, or `remove()` it altogether.
  Unconstrained mutability is prone to error or abuse. Badly designed or 
  buggy code can make it very difficult to identify where and in what
  order state will be read and updated.
 
  This is possible even with a well-defined API like the `ServerFramework`
  example. If a Logger method accidentally reset the permissions to
  only include LOG the error would only appear in a call to the database
  driver performing a completely unrelated aspect of the business logic.

  In the example, what is really needed is a simple, one-way broadcast
  with a single point of assignment in a caller (at 1.) and multiple
  points in called code where the data is consumed (at 3. and 5.).
  This is a very common use case and it is not supported well by the
  more complex API that `ThreadLocal` implements.

  `ThreadLocal` was specified to provide unconstrained mutability in order to
  support a much more general model of communication. Data is able to flow in
  either direction between a caller and called method. This generality may be 
  needed for a few difficult cases. However, in the worst case, programming
  with thread local variables can lead to spaghetti-like dataflow. The result
  is code whose structure is hard to discern, let alone maintain.
   
- *Persistence* — Once set, a `ThreadLocal` is _persistent_: that is to say,
  the incarnation of the value for each thread that calls `set()` is retained
  for the lifetime of the thread, or until a method running on that thread
  calls `remove()`. This is not so much a problem of behaviour as one of
  performance cost.

  It is unfortunately common for developers to forget to remove a `ThreadLocal`,
  or even just to reset it to `null`.  Indeed, for programs that rely on the
  unconstrained mutability of `ThreadLocal` there may be no clear point at
  which it is safe to call `remove()`. This can lead to a long-term memory
  leak. Even though the program may have long since moved on from having
  any use for an object referenced from the `ThreadLocal`, it will not be
  garbage collected unless the thread exits.

  Having to call `remove()` on a `ThreadLocal` to clean it up when it's no
  longer in use is somewhat antithetical to the way that Java usually works.
  It would be better if the context associated with a thread were to be
  cleaned up automatically.

- *Inheritance* — The `ThreadLocal` persistence problem can be much worse
  when using large numbers of threads because`ThreadLocal`s may be inherited
  from parent to child thread. When a `ThreadLocal` `L` is set to a value
  `V` by some Java `Thread` `T` the implementation requires allocating
  some storage private to thread `T` that holds a link from `L` to `V`.  
  If a newly created child `Thread` inherits `L` then it also has to allocate
  its own storage. When an application uses many `Thread`s
  which inherit many `ThreadLocals` this can add significant memory costs. 

  Child threads cannot share the storage used by the parent thread because
  `ThreadLocal`s are mutable. The `ThreadLocal` API requires the parent and 
  child to be able to change their own value without the change being seen
  by the other thread. This is unfortunate because in most cases child threads
  do not `set()` the `ThreadLocals` they inherit.


To sum up, `ThreadLocal` offers more flexibility than is needed by many applications
and comes with some significant costs.

### Complications Due to Virtual Threads

The above problems with thread local variables have become more pressing
in the context of virtual threads. These threads are cheap and plentiful, unlike
today's platform threads which are expensive and scarce.

Platform Threads are:

* Long-running
* Heavyweight
* Pooled

Virtual Threads are:

* Short-running
* Lightweight
* Single-use

It would certainly be useful for these numerous cheap and plentiful
threads to be able to access some context from their parent. For
example, they may share a logger on an output stream. Perhaps they may
share some kind of security policy too.

Because virtual threads are still threads, it is legitimate to for a
virtual thread to carry thread-local variables. The short lifetime of
virtual threads minimises the problem of long term memory leaks via
thread locals. An explicit `remove()` isn't necessary when a thread,
virtual or not, terminates, because when a thread terminates all thread
local variables are automatically removed. However, if you have a
million threads and every one has its own inevitably mutable set of
thread local variables, the memory footprint may become significant.

It would be ideal if the Java Platform provided a way to have per
thread context for millions of virtual threads that is immutable and,
given the low cost of forking virtual threads, inheritable. Because
these ideal per thread variables are immutable, their data can be
easily shared by child threads, rather than copied to child
threads. Thread local variables were the 90s realization of per thread
variables; we need a better realization of per thread vairables for
the modern era.

## Description

An extent local variable is a per-thread variable that allows context
to be set in a caller and read by callees. Unlike a thread local
variable, an extent local variable is immutable: there is no `set()`
method. Context can be anything from a business object to an instance
of a system-wide logger.

The term extent local derives from the idea of an extent in the Java
Virtual Machine. The JVM specification describes an extent as follows:

"It is often useful to describe the situation where, in a given
thread, a given method m1 invokes a method m2, which invokes a method
m3, and so on until the method invocation chain includes the current
method mn. None of m1..mn have yet completed; all of their frames are
still stored on the Java Virtual Machine stack (2.5.2). Collectively,
their frames are called an _extent_. The frame for the current method
mn is called the _top most frame_ of the extent. The frame for the
given method m1 is called the _bottom most frame_ of the extent."

(That is to say, m1's extent is the set of methods m1 invokes, and any
methods invoked transitively by them.)

The value associated with an extent local variable is defined in the
bottom most frame of some extent, and is accessible in every frame of
that extent. The extent local is *bound* to the value.
                                   
###  Example use Of Extent Local Variables

The `ServerFramework` example described above can easily be rewritten to
use class `ExtentLocal` instead of `ThreadLocal`.

     class ServerFramework {
      // 1. Provide a per-thread channel betwen the framework and its components
      final static ExtentLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAdmin() ? LOG|DATABASE : LOG);
        Permissions permissions = new Permissions(p);
        // 2. Bind the extent local for the extent of the run operation
        ExtentLocal.where(PERMISSIONS, permissions)
                     .run(() -> { AppLogic.handleRequest(request, response); });
      }
      ...
    }

    class Logger {
      void warn(String message)  throws PermissionException {
        // 3. Get permissions for request thread and check  
        Permissions permissions = ServerFramework.PERMISSIONS.get();
        if (!permissions.allowed(LOG) {
          throw new PermissionException(...)
        }
        // 4. Ok to write the warning
        write(logFile, "WARNING : %s %s".format(timeStamp(), message));
      }
      ...
    }
    ....

As before, the `ServerFramework` includes an initialization at 1. which
ensures that field `PERMISSIONS` references an `ExtentLocal`. The important
difference occurs at the point where previously there was a call to
`ThreadLocal.set()`. This is changed to use a call to static methods
`where()` and `run()` of class `ExtentLocal`. These two calls work
together to provide the immutable one-way, thread-local communication
that the application needs.

The `where()` call  *binds* `the ExtentLocal` referenced from `PERMISSIONS`
for the extent of the `run()` call. What that means is that a call to
`PERMISSIONS.get()` executed by any method called from `run()` will return
the value provided as argument to `where()`. So, in this case if
`Logger.warn()` gets called, directly or indirectly, by method
`AppLogic.handleRequest()` then the `PERMISSIONS` retrieved at point 3.
will be value passed in the `where()` call at point 2.

It also means that the value retrieved by the `get()` call is specific
to whichever thread is executing the methods. As with `ThreadLocal`, an
`ExtentLocal` has an incarnation that is per thread.

The first big difference is that the binding by `where()` is only visible
from the extent of the code called from `run()`. If a call to
`PERMISSIONS.get()` was inserted after the call to `run()` an exception
would be thrown because `PERMISSIONS` is no longer bound.

The second big difference is that the binding established by `where()` is
immutable within the extent of `run()`. Methods called from `run()` can
`get()` the current binding but there is no `set()` method allowing them
to change the binding established at point 1.

###  Rebinding of Extent Local Variables

The immutability of ExtentLocal bindings means that within a thread a
caller can reliably communicate a single value to the methods it calls.
However, there are times when one of those called methods would
need to communicate a different value to the methods it calls. The
requirement is not to change the original binding but to establish a new
binding for nested calls

As an example consider the `DBDriver` class in the `ServerFramework` example.

    class Database {
      void processQuery(DBQuery query, Consumer<DBRow> rowHandler) {
        // 1. Get permissions for request thread and check 
        Permissions permissions = ServerFramework.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) {
          throw new  InvalidAccessException(...)
        }
        // 2. Run the database query
        List<DBRow> rows = executeQuery(query);
        // 3. Reduce permissions to LOG only
        Permissions reduced = permissions.restrict(LOG);
        // 4. Rebind PERMISSIONS
        ExtentLocal.where(ServerFramework.PERMISSIONS, reduced)
                 .run(() -> 
                     // 5. Parallelize with a fork join executor 
                     try (var s = StructuredExecutor.open()) {
                       for (var row : rows) {
                         // 6. process each row in its own virtual thread
                         s.fork(() -> rowHandler.apply(row));
                       }
                       // 7. wait for all virtual threads to complete
                       s.join();
                     }
                 });
      }

Method `processQuery` runs a `query` against the database to retrieve a
list of results of type `DBRow`.  Argument `rowHandler` is a callback
of type `Consumer<DBRow>` which can be applied to each result by calling
`rowHandler.apply(row)`.  In order to speed up processing of query results
each call to `apply` is executed in its own virtual thread. The handler needs
to be able to log messages but should not attempt to perform any further
database processing. 

The first thing `processQuery` does at 1. is ensure the caller has
the `DATABASE` permission before proceeding to run the query at 2. At 3. it
uses method `restrict` to construct a weaker Permissions object which
removes everything except the `LOG` permission. The call to `where()` at
4 rebinds `PERMISSIONS` to this weaker set of `Permission`s for the extent
of the following `run()` call.

The try at 5. in the lambda passed to `run()` opens a StructuredExecutor.
This class manages collections of virtual threads using a fork join model.
It gets automatically closed at the end of the try with resources block.

Inside the `for` loop at 6. a virtual thread is forked to run the handler on
each `DBRow`. After the loop at 7. a call to `join()` ensures that all the
rows have been processed before the try block is exited.

Rebinding ensures that the callback can only access the behaviour provided
by the `Logger` API. The handler call at 6. is in the extent of the `run()`.
If the handler tries to call a `DBDriver` API method that method will call
`PERMISSIONS.get()`, retrieving the weaker `Permissions` which does not
include `DATABASE`. The rebinding is only local to the `run()` call at 4.
A call to `PERMISSIONS.get()` after the `run()` call completes will still
see the original binding established by the `ServerFranework`.

###  Inheritance of Extent Local Variables

There is one important detail in the `processQuery` code that needs
highlighing. Rebinding of `PERMISSIONS` at 4. happens in the thread that
calls `processQuery`. However, the callback to `rowHandler.apply()` happens
in a forked virtualThread(). How does the `get()` in the forked thread
retrieve the value that was `set()` in the forking thread? This works because
class `StructuredExecutor` has been designed to share `ExtentLocal` bindings
across fork calls. Any `ExtentLocal` bindings present in the thread that calls
`fork()` will be visible to the forked virtual thread.

It is worth emphasising that these bindings really are *shared*. The set of
bindings established by the forking thread can be referenced by the forked
thread without having to be copied.

It is also worth noting that the fork/join model offered by `StructuredExecutor`
means that a value bound in a call to to `where()` has a determinate lifetime,
as far as visibility via the `ExtentLocal` is concerned. The reduced
`Permission` object created at 3. is available for the extent of the
`run()` call at 4. Once that call returns the current thread and its child
threads will no longer need to make use of it.

Using class StructuredExceutor to parallelise row processing means that query
results returning thousands or even millions of rows can be executed in
parallel. If a call to `rowHandler` needs to block, say to log a warning to
a file on disk, work can continue by switching execution to another virtual
thread with almost no overhead. 

## Migrating to extent local variables

In general, for the reasons we've listed above, extent local variables
are likely to be useful in many cases where thread local variables are
used today. We continue to support thread local variables, even with
virtual threads, despite them not being ideal when threads are very
numerous.

The first thing to do is determine whether migrating thread local to
extent local variables is appropriate. If in your application thread
local variables are used in an unstructured way so that a deep callee
`set()`s a `ThreadLocal` which is then retrieved by the caller,
migration may be difficult, and you may find thre is little to be
gained.

However, in many cases extent local variables are exactly what you
need. We've already covered hidden parameters for callbacks in some
depth, but there are other good ways to use extent locals.

- *Extent locals and recursion* — Sometimes you want to be able to
  detect recursion, perhaps because a framework isn't re-entrant or
  because you want to limit recursion in some way. An extentlocal
  variable provides a way to do this: set it once, invoke a method,
  and somewhere deep in the call stack, test again to see if the
  thread-local variable is set. More elaborately, you might need the
  extent local variable to be a recursion counter.

  The detection of recursion is also useful in the case of flattened
  transactions: any transaction started when a transaction in progress
  becomes part of the outermost transaction.


- *The notion of a "current context"*  — Java Concurrency in Practice
  talks about the use of a thread local variable to hold context:

  "... containers associate a transaction context with an executing
  thread for the duration of an EJB call. This is easily implemented
  using a static Thread-Local holding the transaction context: when
  framework code needs to determine what transaction is currently
  running, it fetches the transaction context from this ThreadLocal."

  Another example occurs in graphics, where there is a drawing context.
  Extent local variables, because of their automatic cleanup and
  re-entrancy, are better suited to this than are thread local
  variables.


[Generally, the Gods of Java want you to migrate to extent locals.]


## Where can extent local variables _not_ replace thread local variables?

There are cases where thread local variables are more appropriate than
extent local variables. For example, one popular use of `ThreadLocal`
is to cache objects that are expensive to create. A notorious
example is `java.text.DateFormat`, which is mutable so cannot be
shared between threads without synchronization. In this case, creating
a thread local `DateFormat` object which persists for the lifetime of
the thread might be exactly what you need:

(In hindsight, making `DateFormat` mutable was a mistake, and we'd do
better today, but it was Java 1.1 in 1997. `ThreadLocal` makes it
possible to use this utility class reasonably efficiently in a
multi-threaded program.)

## In summary

Extent locals have the following properties:

* _Locally-defined extent_: The values of extent local variables are only bound
  in the extent of `run()` or `call()`.
* _Immutability_ There is no `ExtentLocal.set()` method: extent locals,
  once bound, are effectively final.
* _Simple and fast code_: In most cases an extent-local `x.get()` is as
  fast as a local variable `x`. This is true regardless of how far
  away `x.get()` is from the point that the extent local `x` is bound.
* _Structure_: These properties also also make it easier for a reader
  to reason about programs, in much the same way that declaring a
  field of a variable `final` does. The one-way nature of the channel makes it much
  easier to reason about the flow of data in a program.

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom_api/jdk.incubator.concurrent/jdk/incubator/concurrent/ScopeLocal.html

## Alternatives

It is possible to emulate many of the features of extent locals with
`ThreadLocal`s, albeit at some cost in memory footprint, runtime
security, and performance.

We have experimented with a modified version of `ThreadLocal` that
supports some of the characteristics of extent locals. However,
carrying the additional baggage of `ThreadLocal` results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or both.
And we'd still have the problem of memory management.

It is better, therefore, not to modify `ThreadLocal` but to give
extent locals a separate identity from thread locals.

## Historical Note
The idea for Extent Locals was inspired by the way many Lisp dialects
provide support for dynamically scoped free variables, in particular
how they behave in a deep-bound, multi-threaded runtime like Interlisp-D.
