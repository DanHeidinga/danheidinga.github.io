---
layout: post
title: Reference.reachabilityFence and JEP 401 value types
published: true
---

`Reference.reachabilityFence(Object ref)` came up in discussion when reviewing the pull request to merge [JEP 401: Value Classes and Objects (Preview)](https://openjdk.org/jeps/401) into the mainline JDK.  The key question: what should a `reachabilityFence` do for an identity-less value type?

To answer that question, we need to understand both what value objects are and what `reachabilityFence` does.

And for the impatient, the TLDR is: a value object has no GC lifecycle to observe, but any identity objects reachable from its fields must still be kept alive.


### Value objects

JEP 401 defines `value objects` as `class instances that have only final fields and lack object identity.` This lack of identity means that two values with the same contents are indistinguishable from each other ("substitutable").  This is great for optimization - the JVM is free to take a value apart into its fields, flow those fields through the method, and reconstitute the instance when needed.  It can do that once, twice, or dozens of times.  The results are always indistinguishable.

But that means value instances don't have a defined lifecycle. The same value object may live in different places during execution - sometimes it may live on the heap, then scalarized in registers on the stack, and then a new indistinguishable heap copy may be created.  It may be possible in JITTED code to throw away all but a single field from the value instance and then only keep that field as being the only field used. When did the instance get GC'd? 

Because that question doesn't have an answer (at least no good one), the JVM doesn't let you ask it.  Any API that is intended to observe the lifecycle of an instance will throw an IdentityException if you attempt to use it for a value object.

From JEP 401:
> The garbage collection APIs in java.lang.ref and java.util.WeakHashMap do not allow developers to manually manage value objects in the heap. Attempts to create Reference objects for value objects throw IdentityException at run time.

and finalizers also don't work:
> The finalize method of a value object will never be invoked by the garbage collector.

There is no API to observe when a value instance is no longer reachable.


### Reference.reachabilityFence

The [javadoc](https://download.java.net/java/early_access/panama/docs/api/java.base/java/lang/ref/Reference.html#reachabilityFence(java.lang.Object)) for `reachabilityFence` states:

> Ensures that the object referenced by the given reference remains strongly reachable, regardless of any prior actions of the program that might otherwise cause the object to become unreachable; thus, the referenced object is not reclaimable by garbage collection at least until after the invocation of this method....
>
> This method establishes an ordering for strong reachability with respect to garbage collection. It controls relations that are otherwise only implicit in a program -- the reachability conditions triggering garbage collection. This method is designed for use in uncommon situations of premature finalization where using synchronized blocks or methods, or using other synchronization facilities are not possible or do not provide the desired control. This method is applicable only when reclamation may have visible effects, which is possible for objects with finalizers (See Section 12.6 of The Java Language Specification) that are implemented in ways that rely on ordering control for correctness.

This means that `Reference.reachabilityFence` ensures that a given object, and the graph of objects reachable through its fields, must be kept alive until the method returns.

Given an object `obj`, that:

* has no Reference objects (like WeakReference or SoftReference or JNI WeakGlobalReference or etc) referring to it,
* doesn't have a finalizer method, and
* doesn't have any other reference fields,

then `Reference.reachabilityFence(obj)` becomes a no-op.  This is a useful property to allow reachabilityFence of escape-analyzed objects to be removed.

One way to think about reachabilityFence(obj) (though not how it's actually implemented), is that it is defined like this:

```
void reachabilityFence(Object obj) {
  // handle any Reference operations pointing at obj
  ....
  // Iterate through all fields and propagate reachability to them
  for(Field f : obj.getClass().allFields()) {
     if (!f.isPrimitive()) {
        reachabilityFence(f.get(obj));
     }
  }
}
```

and that the JIT can inline that operation as deep as it wants.  This is why an escape analyzed object that has a reference field can be transformed from: `reachabilityFence(obj)` to `reachabilityFence(obj.f)` propagating reachability to each of `obj`'s reference fields.

This is a useful mental model for when applying reachabilityFence to value instances.

### Values and ReachabilityFence

If we take the list of conditions for reachabilityFence mentioned above:

> * has no Reference objects (like WeakReference or SoftReference or JNI WeakGlobalReference or etc) referring to it,
> * doesn't have a finalizer method, and
> * doesn't have any other reference fields

We see that the first two conditions don't apply to value instances at all as they don't have the same kind of GC lifecycle.  Does this mean reachabilityFence doesn't apply to them at all?

Not quite.  That third condition still applies.  If we view every value instance as though it is an escape analyzed object, we still need to propagate the reachability through its fields just as we did above.

A JIT compiler can always inline a call of `reachabilityFence(valObj)` so that it becomes:

```
// Iterate through all fields and propagate reachability to them
  for(Field f : valObj.getClass().allFields()) {
     if (!f.isPrimitive()) {
        reachabilityFence(f.get(valObj));
     }
  }
```
Since the JIT will have typically already scalarized the value object into registers, the transformation becomes a call to reachabilityFence for each of the reference fields:

```
reachabilityFence(refField1);
reachabilityFence(refField2);
```

And in many cases where the value object has no reference fields, the reachabilityFence call can be elided.


### Takeaway

Reference.reachabilityFence when applied to value instances must be propagated to the reference fields of that value instance.  For many value instances that only hold primitive fields, this makes the reachabilityFence a no-op.  For value instances with reference fields, the reachabilityFence must be applied to each of those fields.

A small update to C2's handling of reachabilityFence is required to match this model with the work being tracked in [JDK-8386984](https://bugs.openjdk.org/browse/JDK-8386984)

