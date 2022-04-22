# Extent-local variables

## Summary

Introduce extent-local variables to the Java Platform. Extent-local
variables provide a way to share immutable data within and across
threads. They are preferred to thread-local variables, especially when
using large numbers of virtual threads. This is a preview API.

## Goals

- *Ease of use* — Provide a clear programming model to share data
  within and across threads in order to simplify reasoning about
  data flow.
- *Comprehensibility* — Make the lifetime of shared data
  clearly visible in the program code.
- *Robustness* — Ensure that data shared by a caller
   is accessible only to legitimate callees.
- *Performance* — Treat shared data as immutable so as to allow
  sharing among large numbers of threads. Also, this immutability
  greatly helps optimizing compilers.

## Non-Goals

It is not a goal to change the Java Programming Language.

It is not a goal to require migration away from thread-local
variables or to deprecate the `ThreadLocal` API.

It is not a goal for this new programming model to require use
of try-with-resources or interface `AutoCloseable`.

## Motivation

Java programs are often built using libraries of related, reusable
components. For example, a large application may include a server to
handle each incoming request in its own thread, a database driver to provide
persistence, and a logging framework to records progress or noteworthy
event. These components often need to share data between themselves,
independent from the business logic they are combined with.

In this example, as a security measure, the server allocates a
`Permissions` token to each thread that handles a request. Other
server components use that token to restrict access to the
operations they provide.

A thread handling a specific request normally communicates data from
caller to callee via method arguments. However, this is not plainly
not viable when the thread handling the server request calls business
logic code provided by the application developer. The server needs some
other channel to share the `Permissions` token with the driver and logger.

The following diagram provides a snapshot of the server application in
the process of handling two different requests. The server dedicates a
thread to each request for its duration. Request processing starts
with a call to `Server.processRequest()` at 1. The server threads delegate
to the business logic by calling `Application.handleRequest()` at 2. The
business logic eventually tries to use a database connection by calling
DBDriver.open() at 7. At this point DBDriver needs to determine
whether the thread is permitted to access the database.

In Thread 1, the `Permissions` token created by `processRequest()` permit
access to the database. The dashed line indicates a transfer of this token
to the call to `DBDriver.open()` at 7. This method tests the `Permissions`
and proceeds to call `DBDriver.newConnection()` at 8.

In Thread 2, the `Permissions` created by `processRequest()` at 1 do not permit
access to the database. At 7 `DBDriver.open()` detects the error and proceeds to
create an `InvalidPermissionException()` at 8.

    Thread 1                                          Thread 2

    8. DBDriver.newConnection()                       8. InvalidPermissionException()
    7. DBDriver.open() <----------------------+       7. DBDriver.open() <----------------------+
       ...                                    |          ...                                    |
       ...                     Permissions(DATABASE)     ...                       Permissions(NONE)
    2. Application.handleRequest()            |       2. Application.handleRequest()            |
    1. Server.processRequest() ---------------+       1. Server.processRequest() ---------------+

### Currently Supported Options

Developers traditionally turn to thread-local variables in situations
like this. An instance of Class `ThreadLocal` provides developers with
an independent variable for each thread to share its own copy of
data.

The following example shows how a `Server` implementation might use a
thread-local variable to communicate `Permissions` from the server to the database.
`Server.processRequest()` and `DBDriver.open()` implement the methods
invoked at 1. and 7. in the thread call stack diagram above.

    class Server {
      // 1. Provide a per-thread channel between the framework and its components
      final static ThreadLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAuthorized() ? DATABASE : NONE);
        Permissions permissions = new Permissions(p);
        // 2. Set permissions for request thread and run app logic
        PERMISSIONS.set(permissions);
        Application.handleRequest(request, response);
      }
      ...
    }

    class DBDriver {
      DBConnection open(...) throws PermissionException {
        // 3. Get permissions for request thread and check
        Permissions permissions = Server.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) throw new  PermissionException();
        // 4. Ok to connect to a database
        return DBDriver.newConnection(...);
      }
      ...
    }

The `ThreadLocal` declared at 1. in the static field `PERMISSIONS` provides a
way to ensure that each thread handling a request can have its own independent
permissions. This `ThreadLocal` serves as a key that is used to look up
a `Permission` value for the current thread. So, there is not exactly _one_
incarnation of field `PERMISSIONS` shared across all instances of `Server`; instead, there are
_multiple_ incarnations of the field, one per thread, and the incarnation
that is used when code sets or gets `PERMISSIONS`
depends on the thread which is executing the code.

Using a thread-local variable here avoids the need to pass a `Permissions` object as
an argument to calls from the service handler through the business logic
and into the database driver or logger.

The declaration at 1. creates a `ThreadLocal` object and assigns it to static field
`PERMISSIONS`. Before it can be used, the `Server` handler thread must call the `set()`
method at 2. This ensures that the incarnation of field
`PERMISSIONS` specific to the handler thread identifies the permissions
for that request. The server is now ready to execute the business logic.

The database driver calls the `get()` method at 3. to
retrieve the permissions for the current thread. It ensures that the
operation requested by the business logic is permitted before proceeding
to execute it at 4.

This example demonstrates that thread-local variables _can_ be used to implement the
behaviour needed for this case. However, there are problems with the design
of thread-local variables that affect even this example of well-structured use.

- *Unconstrained Mutability*  —  Every thread-local variable is _mutable_:
  that is to say, any code that can access a thread-local variable's `get()` can
  also `set()` it to a different value, or `remove()` it altogether.
  Unconstrained mutability is prone to error or abuse. Badly designed or
  buggy code can make it very difficult to identify where and in what
  order state will be read and updated.

  Such problems are possible even with a well-defined API like the `Server`
  example. For example, if the `DBDriver` accidentally reset the permissions to `NONE`
  the error would only appear the next time the business logic tried to
  perform a database operation.

  The `ThreadLocal` API was _specified_ to provide unconstrained mutability in order to
  support a more general model of communication than we usually need. Data is able to flow in
  either direction between a caller and called method. This generality may be
  needed for a few difficult cases. However, in the worst case, programming
  with thread-local variables can lead to spaghetti-like dataflow. The result
  is code whose structure is hard to discern, let alone maintain.

  In our example, what is really needed is a simple, one-way broadcast
  with a single point of assignment in a caller (at 2.) and one or more
  clear points in called code where the data is consumed (e.g. at 3.).
  This is a very common pattern, and it is not supported well by the
  complex API of `ThreadLocal`.

- *Unbounded Persistence* — Once set, a thread-local variable is _persistent_: that is
  to say, the incarnation of the value for each thread that calls `set()` is retained
  for the lifetime of the thread, or until a method running on that thread
  calls `remove()`.

  Having to call `remove()` on a thread-local variable to clean it up
  when it's no longer in use means that it is all too easy for values to persist
  longer than is desirable. It is unfortunately common for developers to forget
  to remove a thread-local variable.  Indeed, for programs that rely on the
  unconstrained mutability of thread-local variable, there may be no clear point at
  which it is safe to call `remove()`. This can lead to a long-term memory
  leak. Even though the program may have long since moved on from having
  any use for an object referenced from the thread-local variable, it will not be
  garbage collected unless the thread exits.

  Clearly, it would be better if the sharing of data from `set()` and
  `get()` calls could be achieved with a well determined lifetime,
  limiting persistence of the data to a well-bounded interval during
  execution of the thread.

- *Expensive Inheritance* — The thread-local variable persistence problem can be much worse
  when using large numbers of threads because thread-local variables may be inherited
  from parent to child thread. Each thread has to allocate storage for every
  thread-local variable set in that thread. If a newly created child `Thread`
  inherits the thread-local variable, it has to allocate its own storage. When an
  application uses many `Thread`s and they, in
  turn, inherit thread-local variables this can add significant memory costs.

  Child threads cannot share the storage used by the parent thread because
  thread-local variables are _mutable_. The `ThreadLocal` API requires the parent and
  child to be able to change their own value without the change being seen
  by the other thread. This is unfortunate because in most cases child threads
  do not `set()` the thread-local variables they inherit.

The above problems with thread-local variables have become more pressing
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

It would certainly be useful for these numerous, cheap and plentiful
threads to be able to access some context from their parent. For
example, they may share a logger on an output stream. Perhaps they may
share some kind of security policy too.

Because virtual threads are still instances of `Thread`, it is legitimate for a
virtual thread to carry thread-local variables. The problem of long-term
memory leaks via thread locals is less acute in the case of short-lived threads.
An explicit `remove()` isn't necessary when a thread,
virtual or not, terminates, because when it terminates all thread-local
variables are automatically removed. However, if you have thousands
or millions of threads and every one has its own, inevitably mutable, set of
thread-local variables, the memory footprint may become significant.

It would be ideal if the Java Platform provided a way to have per-thread
context for thousands or millions of virtual threads that is immutable and,
given the low cost of forking virtual threads, inheritable. Because
these ideal per-thread variables are immutable, their data can be
easily shared by child threads, rather than copied to child
threads.

It would also be better if the extent of these variables was strongly
and robustly bounded. If it were so, you could open some resource,
share it as context with callees and other threads, and on return
close that resource. You could be certain that all users would have
finished whatever they needed to do, so closing the shared resource
would be safe.

Thread-local variables were the 90s realization of per-thread
variables; we need a better realization of per-thread variables for
the modern era.

To sum up, thread-local variables offers more complexity than is
needed by many applications and come with some significant costs. The
Java Platform needs a way to share context that

* *Is immutable*: has a one-way channel from caller to callee.

* *Has bounded persistence*: once the caller sharing the context
  exits, any shared context can no longer be accessed.

* *Is cheap to inherit*: ideally, sharing context with child threads
  should not add any overhead to thread creation.

## Description

An extent-local variable is a per-thread variable that allows context
to be set in a caller and read by callees. Context can be anything from
a business object to an instance of a system-wide logger. Unlike a
thread-local variable, an extent-local variable is immutable: there is
no `set()`method.

The term _extent-local_ derives from the idea of an extent in the Java
Virtual Machine. The JVM specification describes an _extent_ as follows:

"It is often useful to describe the situation where, in a given
thread, a given method m1 invokes a method m2, which invokes a method
m3, and so on until the method invocation chain includes the current
method mn. None of m1..mn have yet completed; all of their frames are
still stored on the Java Virtual Machine stack (2.5.2). Collectively,
their frames are called an _extent_. The frame for the current method
mn is called the _top most frame_ of the extent. The frame for the
given method m1 is called the _bottom most frame_ of the extent."

That is to say, m1's extent is the set of methods m1 invokes, and any
methods invoked transitively by them.

** The syntax of the `ExtentLocal` API means that the values of
extent-local variables are only shared in the extent of `run()` or
`call()`, and never shared - accidentally or deliberately - outside
that extent.**

It should now be clear that the thread call stack diagram above is
a picture of two separate extents for two different threads.
In both cases the bottom frame of the extent is a call to method
`Server.processRequest()`. In Thread 1 the top frame is a call to
method `DBDriver.newConnection()` while in Thread 2 it is a call to
constructor `InvalidPermissionException()`

The value associated with an extent-local variable is defined in the
bottom most frame of some extent, and is accessible in every frame of
that extent. The extent-local variable is *bound* to the value.

###  Example Use Of Extent-Local Variables

The `Server` example described above can easily be rewritten to
use class `ExtentLocal` instead of `ThreadLocal`.

     class Server {
      // 1. Provide a per-thread channel betwen the framework and its components
      final static ExtentLocal<Permissions> PERMISSIONS = ...;
      ...
      void processRequest(Request request, Response response) {
        int p = (request.isAdmin() ? LOG|DATABASE : LOG);
        Permissions permissions = new Permissions(p);
        // 2. Bind the extent local and run the request handler
        ExtentLocal.where(PERMISSIONS, permissions)
                     .run(() -> { Application.handleRequest(request, response); });
      }
      ...
    }

    class DBDriver {
      DBConnection open(...) throws PermissionException {
        // 3. Get permissions for request thread and check
        Permissions permissions = Server.PERMISSIONS.get();
        if (!permissions.allowed(DATABASE) throw new  PermissionException();
        // 4. Ok to connect to a database
        return DBDriver.newConnection(...);
      }
      ...
    }

As before, the `Server` includes an initialization at 1. which
ensures that field `PERMISSIONS` references an `ExtentLocal`. The important
difference occurs at point 2. where previously there was a call to
`ThreadLocal.set()`. This is changed to use a call to methods
`where()` and `run()` of class `ExtentLocal`. These two calls work
together to provide the immutable one-way, thread-local communication
that the application needs.

The `where()` call  *binds* `the ExtentLocal` referenced from `PERMISSIONS`
for the extent of the `run()` call. That means a call to
`PERMISSIONS.get()` executed by any method called from `run()` will return
the value passed to `where()`. So, in this case, if
`DBDriver.open()` gets called, directly or indirectly, by method
`Application.handleRequest()` then the `PERMISSIONS` retrieved at point 3.
will be value passed in the `where()` call at point 2.

It also means that the value retrieved by the `get()` call is specific
to whichever thread is executing the methods. As with `ThreadLocal`, an
extent-local variable has an incarnation that is per thread.

The first big difference between this example and the previous one
is that the binding established by `where()` is only
accessible within the extent of the code called from `run()`. If a call to
`PERMISSIONS.get()` was inserted after the call to `run()` an exception
would be thrown because `PERMISSIONS` is no longer bound. The syntax
for employing an `ExtentLocal` enforces a well defined lifetime for
data sharing, unlike the unbounded persistence provided by ThreadLocal.

The second big difference from the previous example
is that the binding established by `where()` is
immutable within the extent of `run()`. Methods called from `run()` can
`get()` the current binding but there is no `set()` method allowing them
to change the binding established at point 1.

Note that the `DBDriver` method looks identical. The difference is that
the call to `get()` is invoking a method belonging to `ExtentLocal` not
`ThreadLocal`.

**The properties of immutability and locally-defined extent make it
easier for a reader to reason about programs, in much the same way
that declaring a field of a variable `final` does. The one-way nature
of the channel from caller to callee makes it much easier to reason
about the flow of data in a program.**

** Also, in most cases an extent-local variable `x.get()` is as fast
as a local variable `x`. This is true regardless of how far away
`x.get()` is from the point that the extent-local variable `x` is
bound. **

###  Rebinding of Extent-Local Variables

The immutability of extent-local bindings means that a caller can  use
an `ExtentLocal` to reliably communicate a constant value to the methods it calls.
However, there are times when one of those called methods might
need to use the same extent-local variable to communicate a different
value to the methods *it* calls. The
requirement is not to change the original binding but to establish a new
binding for nested calls.

As an example consider this API method of the framework `Logger`
class.

    public void log(Supplier<String> formatter);

This method formats and prints a log message when logging is enabled.
If logging is disabled it prints nothing. The caller passes in a `Supplier`
argument whose `get()` method is called by `log()` to retrieve the message
text. Using a `Supplier` avoids the cost of formatting when logging is
disabled.

`log()` is a good candidate for the use of rebinding. The `Logger` is called
from business logic code that may have permission to access the database.
However, the `Supplier` that gets executed by the `Logger` only needs to do
text formatting. It should not need to do call any of the components to
do something that requires permissions.
It would be ideal if the extent-local variable `PERMISSIONS` was bound
to an empty `Permissions` instance for the extent of the `formatter.get()` call.

The code for method `log()` is shown below.

    class Logger {
      void log(Supplier<String> formatter) {
        if (loggingEnabled) {
          // 1. Obtain an empty permissions instance
          Permissions nopermission = Permissions.none();
          // 2. Rebind PERMISSIONS for extent of formatter get
          String message = ExtentLocal.where(Server.PERMISSIONS, nopermission)
                                  .call(() -> formatter.get());
          // 3. Print the formatted message
          write(logFile, "%s %s".format(timeStamp(), message));
          ...

The logger obtains an empty `Permissions` instance at point 1.
At point 2. `where()` rebinds extent-local `PERMISSIONS`
to this empty `Permissions` instance for the extent of the `call()` that
follows it. The lambda passed as argument to `call()` invokes `formatter.get()`
to retrieve the formatted message.  If code called from `formatter.get()`
tries to use the database, the permissions check in the `DBDriver` code will
retrieve the empty `Permission` instance, leading to a `InvalidPermissionException`.

Notice that this example uses the paired methods `where()` and `call()`,
whereas the previous example used `where()` and `run()`. That is because in
this example the code executed below `call()` needs to return a result.
The `String` returned by `formatter.get()` is returned from `call()` and used to
bind local variable `message`. `call()` must be passed an argument that returns a
result. `run()` must be passed an argument that does not return a result.

Once again it is clear that the syntax of `ExtentLocal` reinforces the guarantee
of a well defined lifetime for sharing, by limiting rebinding to a nested extent,
i.e. to the lifetime of the call to `formatter.get()`, with no possibility of
changing the binding in the current extent.

###  Inheritance of Extent-Local Variables

There are occasions where it would be useful for an extent-local variable
to be inherited from parent thread to child thread, for much the same
reasons as it is useful to inherit a thread-local variables. This works with
extent-local variables in a way that avoids many of the problems that
arise when using a thread-local variable.

An example use of inheritance is provided by the following method
from the business logic.

    public void processQueryList(List<DBQuery> queries, Consumer<DBQuery> handler);

Method `processQueryList` executes a list of `queries` against the database.
Argument `handler` is a `Consumer<DBQuery>` that is used run each individual
query. That means it can be applied to each`DBQuery` in list `queries` by
calling `handler.apply(query)`.

In order to speed up processing of query results, each call to `apply` is executed
in its own virtual thread.  So, the binding of `PERMISSIONS` for
the caller thread really needs to be inherited by the child virtual thread that
executes `handler`. Inheritance happens automatically because the implementation
of `processQueryList` uses the structured execution framework.

The code for method `processQueryList` is provided below.

      void processQueryList(List<DBQuery> queries Consumer<DBQuery> handler) {
        / 1. Parallelize calls to handler with a structured fork join executor
        try (var s = StructuredExecutor.open()) {
          for (var query : queries) {
            // 2. process each query in its own virtual thread
            s.fork(() -> handler.apply(query));
          }
          // 3. wait for all virtual threads to complete
          s.join();
        }
      }

At 1., `processQuery` uses try-with-resources to
open a `StructuredExecutor`. This is a class provided as part of the virtual
threads implementation which allows a collections of virtual threads to be
managed using a fork join model. It gets automatically closed at the end of
the try with resources block.

Inside the `for` loop at 2. a virtual thread is forked to run the handler on
the current `DBQuery`. After the for loop at 2. a call to `join()` ensures that
all the forked virtual threads have completed. This ensures that all rows
have been processed before the try block is exited.

There is one important detail in the `processQuery` code that still needs
explaining. The call to `handler.apply()` happens in a forked
virtualThread. So, how can a call to `PERMISSIONS.get()` in the forked thread
retrieve the value that was `set()` in the forking thread? This works because
Class `ExtentLocal` has been designed to share bindings across thread forks.
Any extent-local bindings present in the thread that calls `fork()` will be
visible to the forked thread.

It is worth emphasising that these bindings really are *shared*. The forked
thread can reference the set of bindings established by the forking thread
without needing its own local copy.

It is also worth noting that the fork/join model offered by `StructuredExecutor`
means that a value bound in the call to to `where()` has a determinate lifetime,
as far as visibility via the `ExtentLocal` is concerned. The `Permission`
object is available while the child thread is running. The call to `join()`
at the end of the try block ensures that child threads can no longer be using
it. These two capabilities work together to avoid the problem of unbounded
persistence seen when using thread-local variables.

Using class StructuredExecutor to parallelise row processing means that query
results returning thousands or even millions of rows can be executed in
parallel. If a call to `rowHandler` needs to block, say to log a warning to
a file on disk, work can continue by switching execution to another virtual
thread with almost no overhead.

** Extent-local variables are inherited by threads created by a
`StructuredExecutor` with very little overhead. **

## Migrating to extent-local variables

In general, for the reasons we've listed above, extent-local variables
are likely to be useful in many cases where thread-local variables are
used today. We continue to support thread-local variables, even with
virtual threads, despite them not being ideal when threads are very
numerous.

The first thing to do is determine whether migrating thread-local to
extent-local variables is appropriate. If in your application thread
local variables are used in an unstructured way so that a deep callee
`set()`s a thread-local variable which is then retrieved by the caller,
migration may be difficult, and you may find there is little to be
gained.

However, in many cases extent-local variables are exactly what you
need. We've already covered hidden parameters for callbacks in some
depth, but there are other good ways to use extent locals.

- *Re-entrant code* — Sometimes you want to be able to
  detect recursion, perhaps because a framework isn't re-entrant or
  because you want to limit recursion in some way. An extent-local
  variable provides a way to do this: set it once, invoke a method,
  and somewhere deep in the call stack, call `ExtentLocal.isBound()` to see if the
  thread-local variable is set. More elaborately, you might want the
  extent-local variable to be a recursion counter.

- *Nested transactions* — The detection of recursion would also be useful in the case of flattened
  transactions: any transaction started while a transaction is in progress
  becomes part of the outermost transaction.

- *Graphics contexts* — Another example occurs in graphics, where there is a drawing context.
  extent-local variables, because of their automatic cleanup and
  re-entrancy, are better suited to this than are thread-local
  variables.

In general, where the pattern of usage of a thread-local variable
corresponds well with extent-local variables, it's a good idea to
switch to them, for the reasons we've described above.

## Where can extent-local variables _not_ replace thread-local variables?

There are cases where thread-local variables are more appropriate than
extent-local variables. For example, one popular use of thread-local variables
is to cache objects that are expensive to create. A notorious
example is `java.text.DateFormat`, which is mutable so cannot be
shared between threads without synchronization. In this case, creating
a thread-local `DateFormat` object which persists for the lifetime of
the thread might be exactly what you need:

(In hindsight, making `DateFormat` mutable was a mistake, and we'd do
better today, but it was Java 1.1 in 1997. A thread-local variable
makes it possible to use this utility class reasonably efficiently in
a multi-threaded program.)

## API

There is more detail in the Javadoc for the API, at

http://people.redhat.com/~aph/loom_api/jdk.incubator.concurrent/jdk/incubator/concurrent/ScopeLocal.html

## Alternatives

It is possible to emulate many of the features of extent-local variables with
thread-local variables, albeit at some cost in memory footprint,
security, and performance.

We have experimented with a modified version of Class `ThreadLocal` that
supports some of the characteristics of extent-local variables. However,
carrying the additional baggage of thread-local variables results in an
implementation that is unduly burdensome, or an API that returns
`UnsupportedOperationException` for much core functionality, or both.
And we'd still have the problem of memory management.

It is better, therefore, not to modify Class `ThreadLocal` but to give
extent-local variables an extirely separate identity.

## Historical Note

The idea for extent-local variables was inspired by the way many Lisp dialects
provide support for dynamically scoped free variables, in particular
how they behave in a deep-bound, multi-threaded runtime like Interlisp-D.
