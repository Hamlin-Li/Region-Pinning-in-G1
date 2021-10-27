Summary
-------

Support region pinning in G1, i.e avoid disabling GC between the pair of JNI critical functions.


Goals
-----

   - Support *arbitrary* region pinning in G1 full cycle

   - Optimize *arbitrary* region pinning in G1 young/mixed phases

   - Remove GCLocker usage in G1


Motivation
----------

JNI critical functions (e.g `GetPrimitiveArrayCritical`/`ReleasePrimitiveArrayCritical` and `GetStringCritical`/`ReleaseStringCritical`) could provide better performance than corresponding non-critical function, because

> If possible, the VM returns a pointer to the primitive array or string elements; otherwise, a copy is made.

However the benefit comes with a possible cost:

> - There are significant restrictions on how these functions can be used. After calling GetPrimitiveArrayCritical, the native code should not run for an extended period of time before it calls ReleasePrimitiveArrayCritical. We must treat the code inside this pair of functions as running in a "critical region." Inside a critical region, native code must not call other JNI functions, or any system call that may cause the current thread to block and wait for another Java thread. (For example, the current thread must not call read on a stream being written by another Java thread.)    — JNI Guide Chapter 4: JNI Functions

> - These restrictions make it more likely that the native code will obtain an *uncopied* version of the array, even if the VM does not support pinning. For example, a VM may temporarily disable garbage collection when the native code is holding a pointer to an array obtained via GetPrimitiveArrayCritical.    — JNI Guide Chapter 4: JNI Functions

There are 3 ways for critical functions to cooperate with GCs:

   - GCLocker. GCLocker disables GC between `GetXxxCritical` and `ReleaseXxxCritical`. In this way, the client needs to balance whether or not to use the beneficial critical functions.

   - Copying the object back and forth to some area that is not copied (i.e. C heap)

   - Pinning. Pinning means just pin part of the heap, i.e when GC runs, it will skip this pinned part and only work on the rest of the heap. In this way, between the pair of critical functions, GC could still run rather than disabled by GCLocker, and you get the benefit of *uncopied* version of data.

Currently, critical functions cooperate with G1 GC through GCLocker, we expect to support pinning in G1 to replace GCLocker. There are following benefits:

   - Avoid disabling G1 full cycle between the pair of JNI critical functions

   - GCLocker usage could be removed in G1 which will simplify the G1 logic


Description
-----------

Currently, critical functions cooperate with G1 GC through GCLocker, when `GetXxxCritical` is called it will prevent GC from running before corresponding `ReleaseXxxCritical` is called. To support pinning, GC should skip pinned objects, there are 2 ways to pin objects:

   - Pin an object separately. This is complicated to implement in current G1 code base.

   - Pin the region where the object resides in. This is more feasible, as the heap in G1 is region based.

In fact, G1 already fully supports region pinning in principle, e.g humongous objects(regions) and CDS archived regions never moves (pinned), and in full GC it could also pin regions with high liveness ratio. But G1 does not support *arbitrary* region pinning at full cycle, we expect to support it in young, mixed and full GC.

G1 full GC is already ready to support *arbitrary* region pinning(in young or old regions), we just need minor effort to enable it.

In G1 young/mixed GC, there is no active region pinning mechanism, but a passive way to pin a *arbitrary* region already exists, which is evacuation failure. When evacuation fails during young/mixed GC, the object is marked and not moved, and the region it resides in is retained also; later at the post phase of young/mixed GC this object and the region are processed specially. We can reuse this evacuation failure mechanism to implement region pinning. There are 2 parts of work:

   - As mentioned above, current evacuation failure is in passive way, we need to actively apply evacuation failure mechanism to all live objects in a pinned region.

   - Optimize current evacuation failure implementation.

Current evacuation failure handling is not fully optimized in following aspects:

   - Pause time. Currently it uses linear scan to find all evacuation failure regions/objects in post phase.

   - Pinned regions can fill up the Java heap quickly, because current implementation converts all young generation regions that have a single object that failed evacuation into an old generation region. Since they do not have a remembered set, they will only be collected in the next marking cycle.

So, our plan is clear:

   - Optimize existing evacuation failure in young/mixed GC to improve pause time, and try to avoid filling up Java heap quickly

   - Optimize *arbitrary* region pinning implemented through evacuation failure

   - Enable *arbitrary* region pinning in young/mixed GC

   - Enable *arbitrary* region pinning in full GC

   - Besides of these above tasks, we needs to remove GCLocker dependency in G1, at the same time this will also simplify G1 logic.


Alternatives
------------

One other option is to pin an object separately. Pinning single object seems more efficient in space, sounds like it will only pin the space occupied by the object itself. But it's not practical for G1, as all objects in G1 must reside in some region(s), an object cannot exist alone without a region, even a humongous object reside in region(s).

Another option is very similar as our solution, but instead of applying evacuation failure mechanism to all live objects in a pinned region, just pin the specific object and the region it resides in. This is inefficient in 2 ways compared with our solution:

   - We need to record every single pinned objects `Get` by JNI critical functions, and erase the record after the corresponding `Release`.

   - The other unpinned objects in the same pinned region will be moved to new regions, but their original space is still hold by the specific pinned object.

Another option is to implement the *arbitrary* region pinning from scratch in G1. This is not as tempting as our solution, as we don't expect extra benefit (performance, simplicity, …) from this option; on the other hand, it will make the code more complicated.

The last option is to reuse the already existing region pinning mechanism, e.g humongous regions, CDS archived regions, as these regions are not moved (pinned) too. But these regions have totally different attributes from normal regions in young/old generation, they are processed specially in most of G1 phases, it's impractical to extend it to support *arbitrary* region pinning in G1 full cycle.


Testing
-------

Besides of functionality tests, we especially need to do benchmark test or performance measurement to collect performance data.


Risks and Assumptions
---------------------

There is a risk when a lot of regions are pinned because of the JNI critical functions invocation, extreme case is the entire heap is pinned. For this situation, we currently have no solution, but seems the practical application of Shenandoah does not face such issue yet. And the good news is that

   - For most of scenarios, region pinning brings more benefits than issues, as it tries to free clients from balancing between whether or not use JNI critical functions

   - Now clients only needs to balance how many objects they should pin by JNI critical functions.


Dependencies
------------

This JEP is based on several features in G1:

   - G1's region based heap

   - Region pinning in G1 full GC

   - Evacuation failure in G1 young/mixed GC

And it's also based on the fantastic forward-looking work by Thomas Schatzl.
