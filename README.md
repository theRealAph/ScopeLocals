# ScopeLocals

## Rationale

The need for scope locals arose originally from Project Loom. Loom
enables a new style of Java programming where Threads are not a scarce
resource to be carefully managed by thread pools but are much more
abundant, limited only by memory. If we are to be able to create large
numbers of threads &mdash; potentially millions &mdash; we'll need to
make sure that all of the per-thread structures scale well.

ThreadLocals, and in particular inheritable thread locals, are a pain
point in the design of loom. Every time a new Thread instance is
created its set of ThreadLocals (a kind of hash map) is cloned. This
is necessary because a Thread's set of ThreadLocals is, by design,
mutable, so it cannot be shared. For that scalability reason, Loom's
lightweight "Virtual Threads" do not support inheritable ThreadLocals.

However, inheritable ThreadLocals still have a useful role in
conveying context information from parent thread to child, so we need
something which will fill the gap. Ideally we'd like to have a
zero-copy operation when creating a new Thread, so that inheritance
requires only copying a pointer into the new Thread. 



