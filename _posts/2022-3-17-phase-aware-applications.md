---
layout: post
title: 'Static final fields and new deployment models'
---

Java's constantly evolving as a language and a runtime environment. That's one of the amazing things about working on Java. Another area where Java - and let's face it, much of the rest of the ecosystem - is evolving is around deployments. Historically, Java's "sweet spot" has been long running server applications. Today, we're seeing a shift to shorter uptimes due to trends like continous delivery and horizontal scaling. Cloud native approaches are putting pressure on startup time and memory footprint with the wider adoption of containers and kubernetes.

Over the last few years there have been substantial investigations into better deployment models for Java. These new deployment models - like native images and CRIU checkpoint / restore - bring improvements to startup time and memory usage but also new challenges to developing programs.

Existing Java deployments can typically be described as going through three runtime stages:

* **Startup:** characterized by heavy class loading and class initialization
* **Warmup:** characterized by heavy jit compilation while handling the initial application requests
* **Steady-state:** peak performance has been reached, mostly stable in use of memory with reduced jit compilation

The new deployment models augment these existing phases with new phases like "build time", "pre-checkpoint" and "post-restore".

How can a single set of source code tell the runtime that an operation should be lazily initialized on a dynamic VM, build time initialized in a native image, and done prior to the checkpoint when using CRIU?

And why does it matter? Without changing how we express phase-sensitive operations there can be significant performance differences in each scenario.

Making the right choices in these areas will be challenging for application developers and doubly so for library authors.

## Delaying expensive operations
To provide a better experience to users - and more cynically, to claim the best "startup" numbers in benchmarks - developers have looked to identify expensive operations that happen in startup and delay them until first use.

The expensive operations - hopefully as demonstrated by profiles - are changed to use lazy initialization patterns such as the [initialization-on-demand-holder (IODH) pattern](https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom), or simply moving the relevant state to a class that's only initialized during that code path.

Here's an example of using the IODH pattern to delay expensive class initialization operations until first use of the `ExpensiveOp` class:

```java
public class CommonPath {

   private static class LazyInit {
      static final ExpensiveOp instance = new ExpensiveOp();
   }
   
   public static ExpensiveOp getExpensiveOp() {
   	   return LazyInit.instance;  // trigger initialization of ExpensiveOp class
   }
}

class ExpensiveOp {
   static {
      // slow operations that aren't needed by every user occur here
   }
}
   
```

The greatest challenge in refactorings to adopt these patterns is dealing with the `static final` fields as they can only be set by a class initializer (`<clinit>` method) to maintain their `final` nature.

## Static final fields
Static final fields are great! They provide invariants such as being set exactly once - making them easier to reason about - and don't need additional locking protocols as they benefit from the VM-enforced single threaded execution of the `<clinit>` method. And the JIT can often optimize code using `static finals` as it knows the value won't change (this is mostly true).

Moving operations that set `static final` fields out of startup shouldn't cost developers these benefits.

And it doesn't have to as long as there's an appropriate class where the initialization of the field can be moved - either an existing class that is already only initialized on the specialized code path or by introducing another class to hold the field (ie: IODH pattern).

This works. It's done today for the most egregious operations impacting startup - those expensive operations that are only used by a small set of less common use cases.

So what does this have to do with new deployment models?

## New deployment models
There has been a lot of exploration in the Java ecosystem around different deployment models.  

GraalVM has produced native images which "close" over the application and, in exchange for reducing Java's dynamic nature, provide faster startup by pre-loading all classes and selectively initializing them at build time.

CRIU experiments are happening in various parts of the ecosystem including Eclipse OpenJ9's [criu project](https://github.com/eclipse-openj9/openj9/labels/criu), in OpenJDK's [Project CRaC](https://openjdk.java.net/projects/crac/), and even in Red Hat's own [Jigawatts project](https://github.com/rh-jmc-team/jigawatts). All of these prototypes allow an application to be started before being checkpointed: basically paused and saved to disk. The checkpoint can then be restored later bringing the application back to life. Of particular interest is taking checkpoints during the CI/CD build pipeline, saving startup and initial framework loading time, and then restoring them during deployment for a faster time to peak performance.

These new deployment models intersect with `static final` fields when it comes to fast startup as there are now more choices on *when* to schedule an operation:

* at build time
* at run time
* before a checkpoint
* after a restore
* as lazily as possible

## When should expensive operations run?
Should that expensive operation, rooted in a `static final`, be run as lazily as possible (ie: good for a dynamic JVM) or should it be scheduled to run at build time (in a native image) or before the checkpoint occurs (in a CRIU workload)?

As an example of why the right scheduling matters: the OpenJ9 project did some experiments with a simple Open Liberty application and found they could produce a 10+% performance difference depending on when a CRIU checkpoint was taken during the application run. Taking the checkpoint after first request, rather than after loading and initializing the framework, resulted in significantly better performance immediately on restore. Unfortunately, the later a checkpoint is taken the less portable it is as it will have recorded more of the host machines environment and settings into the generated code and tuned itself to that machine.  Moving it to another machine will require more fixups and makes restoring it a greater challenge. Further, running the first request increases the complexity of the CI/CD build pipeline substantially compared with simply starting and stopping the framework.

All the lazy initialization work done for running on the dynamic JVM  - including lazily setting instance fields - is now at odds with benefiting from checkpoints. To benefit, most of those lazy operations should be scheduled as early as possible so the classes and calculations are done before the checkpoint. 

Native images experience similar issues with preferring that code be written to allow build time initialization. Code using lazy initialization may not be runnable during build time and therefore must incur the runtime costs of the operation when it would otherwise have been valid to execute the operation at build time.

## Developers: start thinking about requirements now
While these new deployment models are in development, it's time to look up from your regular scheduled problem solving to think about how your application will evolve in the next few years.

What are the biggest pain points for your deployments today? Slow startups? Runtime memory usage? Performance of the first request? Or something else entirely? What ever it may be, you need to start thinking about the tradeoffs you're willing to make to stay on the dynamic JVM or move to one of these new deployment models.

There's no free lunch - the complexity is inherent in the system. The question is where you shift it to: at runtime, at build time, or in the CI/CD pipeline.

The good news for application developers is that your application will be deployed (and therefore developed) with one model in mind out of - dynamic JVM, native image, checkpoint/restore - and that's the only one you need to care about. That's the only target you need to optimize.

While it's good news you likely get to target one deployment model, there will still be a steep hill to climb when switching your existing application from deploying on the dynamic JVM to one of these new models.

## Libraries & frameworks: new challenges ahead
Library and framework developers, sorry but your lives just got (even) harder. When these new models gain popularity, you'll need to decide whether to support them (or not) and deal with the requests from users to support those you currently don't. It means an increased testing load and more difficult performance tuning.

And `final static` fields will be on the forefront of this challenge. Determining when they get initialized in each of the models is going to be tough. Splitting a library into different versions for each supported model will be a maintenance nightmare. So with one source base you'll need to address all three deployment options.

Choosing to only support one, or two, of the new models may split the ecosystem. This is kind of remicent of concerns around the introduction of the Java Platform Module System splitting the ecosystem - those concerns haven't come to fruition yet due to the slow(ish) uptake of JPMS. The improvements from the new deployment models are more likely to drive quicker adoption leading to increased risk in this area.

## No good solutions yet
Unfortunately, there really aren't good solutions for supporting all models yet. 

GraalVM has Substitutions to help adapt libraries when you can't upstream the change but they're too blunt a tool for most use and, like carrying any patches external to the upstream, risk divergence and maintenance burdens.

Various proposals suggest adding `isBuildTime()` or `isCheckpointMode()` APIs but all increase the maintenance and testing burdens. And introduce multiple ways for operations to occur.

```java
class Holder {
  static final Object state = someExpensiveOperation();
}

class CommonPath implements Resource {
  void someUncommonOperation() {
    UncommonPath.execute(Holder.state); // (1) lazy runtime init
  }
  
  static {
    if (System.isBuildTime()) {
      Holder.state;  // (2) force build time init
    } else if (System.isCheckpointMode()) {
      Core.getGlobalContext().register(new CheckpointHelper()); // (3a) register checkpoint hook
    }
  }
  ... rest of CommonPath impl ...
}
 
class CheckpointHelper implements Resource {
  void beforeCheckpoint(Context c) {
    Holder.state; // (3b) force pre-checkpoint init 
  }
}  
```

Code `(1)` that used lazy initialization for startup improvements on a dynamic JVM is now coupled with extra maintenance duties to support `(2)` build time init in native images or pre-checkpoint init `(3ab)` with CRIU.  

There used to be one way to "say what you mean" and it corresponded to "please delay this operation until it's used". Now there more degrees of freedom but we don't yet have the tools to express them in the language, or the guidance on what to express for different cases.

## Conclusions
Having our cake and eating it too is hard. New deployment models give faster startup but complicate the lives of library and framework developers.

We need the language to let us say what we mean, to support new deployment models, and to avoid the maintenance burdens of having multiple ways initial `static final` fields.

And we need to be clear on what we mean - does "initialize this lazily" mean must be done as late as possible?  Or just don't do it on my critical path? Or do the opposite if not running on a dynamic JVM?

Lots more work to do in this space!

----

(Many thanks to reviewers for comments and suggestions: Andrew Dinn, Vijay Sundaresan, and Ben Evans)
