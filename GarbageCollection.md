# Garbage Collection

Garbage collection means we don't have to manually free memory but it does not prevent memory leaks.

The CLR garbage collector (GC) is an almost-concurrent, parallel, compacting, mark-and-sweep, generational, tracing GC.

## Mark and Sweep

**Mark**: identify all live objects

**Sweep**: reclaim dead objects

**Compact**: shift live objects together

## Roots

Starting points for the garbage collector. It includes:
* Static variables
* Local variables
* Finalization queue
* F-reachable queue
* GC Handles etc

## Workstation GC

Workstation GC is the deafult for almost all .NET applications. GC runs on a single thread.

Concurrent workstation GC is a special GC thread. CLR creates a special GC thread as soon as application starts running. This thread monitors the GC heap and performs garbage collection ocassionally. It runs concurrently with application threads, which means it produces only short suspensions. Workstation GC does not use all CPU cores.

## Server GC

One GC thread per logical processor, all working at once. Server GC also has separate heap area for each logical processor in order to be faster.

It is available only on multiprocessor server systems.

## Switching GC Flavors

Configure preferred flavor in app.config

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <runtime>
        <gcServer enabled="true|false"/>
        <gcConcurrent enabled="true|false"/>
    </runtime>
</configuration>
```

## Generational Garbage Collection

Divide the heap into regions and perform small collections often.
**New** objects die fast, **old** objects stay alive.

Three heap regions (generations)
* Gen 0
* Gen 1
* Gen 2 - does not have a size limit

Gen 0 and Gen 1 are typically quiet small. Depends on the CPU cache size, it can be a few MBs.
Survivors from Gen 0 are promoted to Gen 1 and survivors from Gen 1 are promoted to Gen 2.

Make sure that temporary objects die young and avoid frequent promotions to Gen 2.

## The Large Object Heap

Large objects are stored in a separate heap region (LOH). Large means larger than 85.000 bytes or array of >1000 doubles.

The GC does not compact the LOH and this may cause fragmentation.
The LOH is considered part of Gen 2.

## Performance Problems with Finalization

Finalization extends object lifetime by putting them on F-reachable queue. The F-reachable queue might fill up faster than the finalizer thread can drain it. This can be solved by deterministic finalization on application thread by using Dispose.

## The Dispose Pattern

Stay away from finalization and use deterministic cleanup. No performance problems but you are responsible for resource management.

# Reducing pressure on the Garbage Collector

## Switching to Value Types

Value types are more friendly to the GC and they are smaller than reference types. Sync Block Index and Method Table Pointer create overhead of 16 bytes on x64 systems.

|                  |                  |                      |       |       |
|------------------|------------------|----------------------|-------|-------|
| **class** Point  | Sync Block Index | Method Table Pointer | int x | int y |
| **struct** Point | int x            | int y                |       |       |

Value types on the heap are embedded in their container so it makes it easier for the GC to traverse. Consider using value types in performance sensitive code.

## Reducing Allocations

More allocations mean more work for the GC and allocating many large objects is especially bad.

**Buffering** can reduce small object allocation (most famous example is using StringBuilder for string concatenation).

**Pooling** can reduce large object allocation.
