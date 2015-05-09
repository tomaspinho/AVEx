Definitions:

In computer science, **reference counting** is a technique of storing the number of references, pointers, or handles to a resource such as an object, block of memory, disk space or other resource. It may also refer, more specifically, to a garbage collection algorithm that uses these reference counts to deallocate objects which are no longer referenced. Examples of languages whose **Garbage Collector** uses this method: **Perl**, **PHP**, **Python**.

In computer programming, **tracing garbage collection** is a form of automatic memory management that consists of determining which objects should be deallocated ("garbage collected") by tracing which objects are reachable by a chain of references from certain "root" objects, and considering the rest as "garbage" and collecting them. Tracing garbage collection is the most common type of garbage collection. Examples of languages whose **Garbage Collector** uses this method: **Java** and all other languages implemented on the **JVM**: **Closure**, **Scala**, etc.

A free list is a data structure used in a scheme for dynamic memory allocation. It operates by connecting unallocated regions of memory together in a linked list, using the first word of each unallocated region as a pointer to the next. It is most suitable for allocating from a memory pool, where all objects have the same size. Free lists make the allocation and deallocation operations very simple. To free a region, one would just link it to the free list. To allocate a region, one would simply remove a single region from the end of the free list and use it. If the regions are variable-sized, one may have to search for a region of large enough size, which can be expensive. A free list with a fixed cell size may suffer internal fragmentation, because objects may be smaller than the cell size.
 
Method                     | Advantages | Disadvantages
---------------------------|------------|--------------
Reference Counting         | Objects can be reclaimed as soon as they can no longer be referenced without long pauses for collection cycles and with clearly defined lifetime of every object. Simple to implement. Performance does not deteriorate as the total amount of free space decreases. | The frequent updates it involves are a source of inefficiency. Requires every memory-managed object to reserve space for a reference count. The naive algorithm can't handle reference cycles - objects that refer to themselves - which means there needs to be a backup tracing collector.
Tracing Garbage Collection | Does not require frequent updates, as it is complete on its own run. Can handle reference cycles. | Objects are only reclaimed when it is run, as its trigger is implementation dependent. Takes longer due to its completeness. Performance deteriorates as total amount of free space decreses - because of internal tree-like structure for traversing allocated objects - and as number of objects increase - more nodes in same tree.

The paper identifies and solves the remaining problems, as the authors (Shahriyar and Blackburn) had presented a paper in 2012 identifying and solving a few problems related to RC. Current implementations (before the papers) were 30% (or more) slower than high performance tracing collectors.

In a previous paper, the authors identified programs usually have the following characteristics: **the vast majority of reference counts are low (<5)** - so the RC uses only a few bits for reference counting (sticks overflowing counts at max and corrects when it traces heap during cycle collection - backup tracing) - **many reference count increments and decrements are to newly allocated objects** - new objects are allocated as "dead", which eliminates useless work. The result was a reference counting collector with the same performance as a whole-heap tracing collector - but within 10% of the best implementation in MMTk.

This paper solves the previous inefficiency by introducing RC Immix, a version of the algorithm that, instead of using free-lists for object allocation - a source of inefficiency due to problems with cache locality and deallocating each object independently (identified in previous paper by Blackburn and Mckinley), uses allocated contiguous blocks and places objects created consecutively in time consecutively in space. Garbage collection is done in lines or blocks instead of individual objects by having reference counters to all objects in a line.
RC Immix compacts newly allocated objects and eliminates memory fragmentation by moving objects around lines and blocks.

Figure 1. Immix Heap Organization

Table 1. The mutator characteristics
Semi-space is the canonical example of a contiguous allocator and the limit point, but it is incompatible with reference counting. Divides heap in two halves, the live objects are copied into the other and the first half is freed.

The second advantage of contiguous allocation is that it uses fewer instructions per allocation, principally because it zeros free memory in bulk using substantially more efficient code. The allocation itself is also simpler because it only needs to check whether there is sufficient memory to accommodate the new object and increase the bump pointer, while the free-list allocator has to look up and update the metadata to decide where to allocate. 

Collins’ first reference counting algorithm suffered from significant drawbacks including: a) an inability to collect cycles of garbage, b) overheads due to tracking very frequent pointer mutations, c) overheads due to storing the reference count, and d) overheads due to maintaining counts for short lived objects. To solve this, the authors implemented a few optimizations, present in other literature: 

**Deferral** - Deferred reference
counting ignores mutations to frequently modified variables, such as those stored in registers and on the stack. Deferral requires a two phase approach, dividing execution into distinct mutation and collection phases. This requires the usage of stack maps, which may be difficult to implement, but are worth the performance increase.

**Coalescing** - A reference counting algorithm would typically execute rc(O1)--, rc(O2)++, rc(O2)--, rc(O3)++, rc(O3)--, ..., rc(On)++. In order to have the reference count properly evaluated at the end of the interval (between runs of the Garbage Collector) it is enough to perform rc(o1)-- and rc(on)++. Therefore, if we log the first mutation, we can properly perform the updates without doing all inbetween.

**Limited Bit Counts** - As shown before by the authors, the reference count can use fewer bits.

**Cycle collection** - To attain completeness, a separate backup tracing collector executes from time to time to eliminate cyclic garbage.

**Young Objects** - As the weak generational hypothesis states, most objects die young [23, 30], and as a consequence, young objects are a very important optimization target. The authors choose to allocate them as "dead" and only mark them live when their reference is found in other objects.

Immix tackles fragmentation using opportunistic defragmentation, which mixes marking with copying. At the beginning of a collection, Immix identifies fragmentation as follows: blocks with available memory indicate fragmentationbecause although available, the memory was not usable by the mutator. Furthermore, the live/free status for theseblocks is up-to-date from the prior collection. In this case, Immix performs what we call here, a reactive defragmenting collection. To mix marking and copying, Immix uses two bits in the object header to differentiate between marked and forwarded objects. At the beginning of a defragmenting collection, Immix identifies source and target blocks. During the mark trace, when Immix first encounters an object thatresides on a source block and there is still available memory for it on a target block, Immix copies the object to atarget block, leaving a forwarding pointer. Otherwise Immixs imply marks the object as usual. When Immix encounters forwarded objects while tracing, it updates the reference accordingly. This process is opportunistic, since it performs copying until it exhausts memory to defragment the heap.

As mentioned before, each object is born dead in RC, with a zero reference count to elide all reference counting work for short lived objects. In RC Immix, each line is also born dead with a zero live object count to similarly elide all line counting work when a newly allocated line only contains short lived objects. RC only increments an object’s reference count when it encounters it during the first GC after the object is born, either directly from a root or due to an increment from a live mutated object.

During a reference counting collection, before RC Immix increments an object’s reference count, it first checks the new bit. If the object is new, RC Immix clears the new object bit, indicating the object is now old. It then increments the object reference count and the live object count for the line. When all new objects on a line die before the collection, RC Immix will never encounter a reference to an object on the line, will never increment the live object count, and will trivially collect the line at the end of the first GC cycle.

Figure 2

Cycle Collection
RC Immix relies on a backup tracing cycle collector to correct incorrect line counts and stuck object counts. It uses a mark bit for each object and each line During cycle collection, the collector marks each live object, marks its corresponding line, and increments the live object count for the line when it first encounters the object. At the end of marking, the cycle collector reclaims all unmarked lines

Proactive Defragmentation
RC Immix copies as many surviving new objects as possible given a particular *copy reserve*. This *copy reserve* is set aside by the allocator during normal program execution and is defined by a *copy reserve heuristic* chosen by the authors. MAX is such a purposed heuristic and takes the maximum survival rate of the last N collections (in lines).

Reactive Defragmentation
RC Immix also defragments at the start of each cycle collection based on fragmentation level thresholds by evaluating free blocks and available partially filled blocks. When an object is moved, a forwarding pointer is left on the previous position. During collection, if a reference pointer to an object with a forwarding pointer is found, it is replaced by the new address and the pointer marked as free.

Result methodology
Jikes RVM does not have a bytecode interpreter. Instead, a fast template-driven baseline compiler produces machine code when the VM first encounters each Java method. The adaptive compilation system then judiciously optimizes the most frequently executed methods. To reduce perturbation due to dynamic optimization and to maximize the performance of the underlying system that we improve, we use a warmup replay methodology. Before executing any experiments, we gathered compiler optimization profiles from the 10th iteration of each benchmark. When we perform an experiment, we execute one complete iteration of each benchmark without any compiler optimizations, which loads all the classes and resolves methods. Operating System. We use Ubuntu 10.04.01 LTS server distribution and a 64-bit (x86 64) 2.6.32-24 Linux kernel. Hardware Platform. We report performance, performance counter, and detailed results on a 32nm Core i7-2600 Sandy Bridge with 4 cores and 2-way SMT running at 3.4GHz. The two hardware threads on each core share a 32KB L1 instruction cache, 32KB L1 data cache, and 256KB L2 cache. All four cores share a single 8MB last level cache. A dualchannel memory controller is integrated into the CPU. The system has 4GB of DDR3-1066 memory installed.

Results
Tested garbage collectors:  1. GenImmix, which uses a copying nursery and an Immix
mature space. 2. Sticky Immix, which uses Immix with an in-place generational adaptation [6, 15]. 3. Full heap Immix. 4. RC from Shahriyar et al. 5. RC Immix (no PC) which excludes proactive copying and performs well in moderate to large heaps due to very low collection times. 6. RC Immix as described in the previous section, which performs well at all heap sizes.
We normalize to GenImmix since it is the best performing in the literature [6] across all heap sizes and consequently is the default production collector in Jikes RVM

Figure 3

Figure 4

(Read the graph and provide conclusions)

This graph makes it clear that the underlying heap structure has a profound impact on mutator (application) performance.

Further analysis and opportunities

**Reference Level Coalescing** - Attempt to register the rememberance bit of Coalescing in the reference, instead of on the object. The authors have attempted this (by stealing the higher bit in the reference) and obtained a slowdown.

**Conservative Stack Scan** - Spend less time scanning the stack for references to objects by building a root object stack map. May prove faster in future works.

**Root Elision** - Instead of always traversing roots to find alive objects and references when doing backup tracing, mantain a shadow map of globals and only traverse the ones that have been changed since the last GC run.
