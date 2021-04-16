# ScopeLocals
## Rationale

The need for scope locals arose originally from Project Loom. Loom
enables a new style of Java programming where Threads are not a scarce
resource to be carefully managed by thread pools but are much more
abundant, limited only by memory. If we are to be able to create large
numbers of threads &mdash; potentially millions &mdash; we'll need to
make sure that
