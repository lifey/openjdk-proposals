# Ergonomics Profiles

Ongoing PR: https://github.com/microsoft/openjdk-jdk/pull/9

## Summary

Introduce a new JVM feature for Ergonomics Profiles, with a `shared` profile for the existing heuristics and a `dedicated` option for when the JVM is running on systems with dedicated resources (memory and CPU) for the one JVM process.

## Goals

1. Introduce the concept of ergonomics profiles.
2. Introduce a new flag to select an ergonomic profile.
3. Define existing heuristics and ergonomics as `shared`.
4. Introduce a `dedicated` profile designed for systems where the JVM is the dominant process using resources, e.g., in a canonical container deployment scenario. The purpose of this profile is to utilize most of the resources allocated to the machine/container.  
5. Automatically select the `dedicated` profile when the JVM believes the system has dedicated resources to it.
6. The JVM will make a better effort to remain withing the total memory limit. 


## Non-Goals

This JEP does not aim to increase performance of applications when running under the `dedicated` profile.

## Success Metrics

No specific success metrics are needed.

## Motivation

The original design of default JVM ergonomics and heuristics was aimed at traditional bare metal servers or large virtual machine environments where the JVM had to share resources with other processes (e.g., a data store) running on that same environment. Today, more than half of Cloud based JVM workloads are running in dedicated environments.

A study in 2023 by New Relic, an APM vendor with access to millions of JVMs in production, identified that more than 70% of their customers' JVMs were running inside environments with resources dedicated to the JVM (i.e., in containers). Many of these JVMs with dedicated resources were running without explicit JVM tuning flags. Therefore the JVM was running with default ergonomics traditionally aimed at shared environments. Under this condition, the JVM does not utilize most of the memory available, and the workload wastes resources. Users then tend to resort to horizontal scaling to address performance issues before addressing resource planning for the JVM (and its tuning). This premature move to horizontal scaling, in turn, leads to more resource waste, both in terms of computing resources and engineering resources with hours spent in monitoring, configuring, and operationalizing the deployments, followed by manual tuning).

With an increase, by default, of resource consumption in environments with dedicated resources for the JVM process, the JVM has more opportunities to improve throughput and latency, or at the least, meet the resource consumption (footprint) expected by the user. 
On the other hand allocating too much resources to the JVM can be dangerous. Allocating too many CPU threads to GC can hur performance and even more dangerous breaching the container memory limit will cause the OOMKiller to kick in and brutally kill the JVM process. 

Currently, the default ergonomics of the HotSpot JVM are:

### Memory Allocation

The default maximum heap size of the JVM varies from 50% to 25%, depending on how much memory is available in the environment. The table below describes the current heuristics of OpenJDK 21:

**_Max heap size_**

| Available memory | Default |
|------------------|---------|
| < 256 MB         | 50%     |
| 256 - 512 MB     | ~126 MB |
| >= 512 MB        | 25%     |

The initial heap size also varies. The table below describes the current heuristics of OpenJDK 21:

**_Initial heap size_**

| Available memory | Default      |
|------------------|--------------|
| <= 512 MB        | 8 MB         |
| >= 512 MB        | 1.5625%-1.7% |

**_Min heap size_**

| Available memory | Default      |
|------------------|--------------|
| 64 MB – 8192 MB  | 8 MB         |

We can observe that these amounts don't adequately map to the intended resource plan of dedicated environments. The user may have already considered allocating, e.g., 4GB of memory to the environment and expect the JVM to use _nearly_ 4GB of memory. In this example, the JVM will instead use only 1GB of memory by default (25%) for its heap, and the user will have to manually tune the JVM if they want to ensure larger heap.

Furthermore, it is likely that the JVM can reclaim heap memory later than it does in shared environments, as the JVM is the only process running on such environment. Knowing it is the only or primary process, this means that the JVM may set the initial heap size closer to the maximum heap size while having a suitable minimum heap size for other memory pools (e.g., native memory).

### Garbage Collector

The default garbage collector selection happens only among two: Serial GC and G1 GC, based on the number of active processors seen by the JVM, and the amount of available memory, with a slightly different way of defining the GC thread pool:

**_GC selection_**

| Memory     | Processors | GC selected |
| ---------- | ---------- | ----------- |
| \>=1792 MB | \>1        | G1          |
| Otherwise  |            | Serial      |

Which GC is used by the application will impact the amount of native memory consumed by the JVM, and the throughput and latency of the application.

_**GC threads**_

We document the current implementation detail of GC threads ergonomically configured by the JVM for reference.

| GC        | \# of CPUs | ConcGCThreads                   | ParallelGCThreads      |
| --------- | ---------- | ------------------------------- | ---------------------- |
| Serial GC | any        | 0                               | 0                      |
| G1 GC     | 1-8        | max((ParallelGCThreads+2)/4, 1) | ceil(#CPUs)            |
| G1 GC     | \>8        | max((ParallelGCThreads+2)/4, 1) | 8 + (#CPUs-8) \* (5/8) |

## Description

We propose adding the concept of Ergonomics Profiles while naming the existing default ergonomics as `shared` and adding a second profile called `dedicated` for when the JVM is to run on environments with dedicated resources, such as, but not limited to, containers.

### Selecting a profile

To select a profile, this JEP proposes a new flag:

```sh
-XX:ErgonomicsProfile=<shared|dedicated>
```

Users may also select a profile by setting this flag in the environment variable `JAVA_TOOL_OPTIONS`. This will be useful for VM templates meant for dedicated JVM-based applications.

### Default profile selection

The `shared` ergonomics profile will be selected by default unless the JVM believes it is running in a dedicated environment (e.g., containers, cgroups, zones). The JVM will activate the `dedicated` profile in this case.

### Shared profile

The `shared` profile represents the heuristics that the HotSpot JVM uses today.

### Dedicated profile

 Assuming the environment is dedicated to the JVM, the `dedicated` profile will contain different heuristics to maximize resource consumption. This profile will maximize heap size allocation, optimize garbage collector selection with a more extensive set of options and considerations, and more aggressively estimate native memory consumption.

The table below describes what the `dedicated` profile will set for the JVM:

* GC selection: see below
* Default maximum heap size: see below
* Default initial heap size: 50%
* Default minimum heap size: 25%
* GC threads: unchanged

#### Selecting a garbage collector in dedicated profile

| Memory     | Processors | GC selected |
| ---------- | ---------- | ----------- |
| Any        | 1          | Serial      |
| \<=2048 MB | \>1        | ParallelGC  |
| \>2048 MB  | \>1        | G1          |
| \>=16 GB   | \>1        | ZGC         |

* Memory refers to the physical machine excluding swap space on a physical/virtual machine and the cgroup memory limit in the case of a container. 

#### Default min/max heap size

We progressively grow the heap size percentage based on the available memory to reduce waste on memory reserved for off-heap (native) operations. This heap size percentage growth is an estimate that native memory usage is required at occasional moments throughout the operation of the JVM for most applications. Otherwise, if the percentage were to be the same, progressive waste would exist. Users are still encouraged to monitor and observe memory consumption and adjust heap size allocation accordingly.

| Memory      | Min Heap size |
| ----------- |---------------|
| <  0.5 GB   | 50%           |
| >= 0.5 GB   | 75%           |
| >= 4   GB   | 80%           |
| >= 6   GB   | 85%           |
| >= 16  GB   | 90%           |



#### Changes in the Mechanism to increase heap size 
Introduce another internal Parameter in the JVM effective_max_heap_size. When the JVM is about to increase the JVM heap size it 
will not be able to pass this value.When the JVM starts effective_max_heap_size = min heap size. After the JVM stabilizes
effective_max_heap_size = MemSize - All Non Heap Memory Size - slackMB
slackMB = 50M and is configurable 

once effective_max_heap_size is set the heap cannot be increased anymore in addition native memory allocations will be blocked which slackmemory is depleted. 
this includes: new threads, loading classes and code cache optimizations. The purpose of this is to prevent an OOMKIll at all cost.  

### API to identify selected profile

An application can obtain the profile selection programmatically by reading the property `java.vm.ergonomics.profile`:

```java
var ergonomicsProfile = System.getProperty("java.vm.ergonomics.profile");
```

The profile selection may also be obtained programmatically through JMX by the inclusion of the following method in `java.management.RuntimeMXBean`:

```java
String getJvmErgonomicsProfile()
```

The value may also be obtained through the `MBeanServerConnection.getAttribute` method, with the name `JvmErgonomicsProfile`:

```java
MBeanServerConnection mBeanServerConnection = ...

var objectName = new ObjectName(ManagementFactory.RUNTIME_MXBEAN_NAME);
var ergonomicsProfile = mBeanServerConnection.getAttribute(objectName, "JvmErgonomicsProfile");
```

## Alternatives

Currently, the only alternative to this proposal is manually tuning the JVM. This exercise is often complex, challenging, and time-consuming for Java developers.

## Testing

// What kinds of test development and execution will be required in order
// to validate this enhancement beyond the usual mandatory unit tests?
// Be sure to list any special platform or hardware requirements.

We will provide tests to ensure the correct profile selection on expected environments:

 1. Execution in a VM shall select the `shared` profile
 1. Execution in a container shall select the `dedicated` profile

We will provide other tests to validate the heuristics of the `dedicated` profile.

## Risks and Assumptions

Allocating too much memory for the process can cause the container to be killed. 

## Dependencies

We list of related issues, in no particular order, that we may address for, or due to, this JEP:

1. [JDK-8261242: \[Linux\] OSContainer::is_containerized() returns true when run outside a container](https://bugs.openjdk.org/browse/JDK-8261242)
1. [JDK-8302744: Refactor Hotspot container detection code](https://bugs.openjdk.org/browse/JDK-8302744)
1. [JDK-8264482: container info misleads on non-container environment](https://bugs.openjdk.org/browse/JDK-8264482)
1. [JDK-8285277: Improve container support for memory limits](https://bugs.openjdk.org/browse/JDK-8285277)
1. [JDK-8292742: Remove container support method cpu_share()](https://bugs.openjdk.org/browse/JDK-8292742)
1. [JDK-8284900: Check InitialHeapSize and container memory limits before startup](https://bugs.openjdk.org/browse/JDK-8284900)
1. [JDK-8278492: Confusion about parameter -XX:MinRAMPercentage](https://bugs.openjdk.org/browse/JDK-8278492)
1. [JDK-8287185: Consolidate HotSpot and JDK tests for CgroupSubsystemFactory](https://bugs.openjdk.org/browse/JDK-8287185)
1. [JDK-8078703: Ensure that GC's use of processor count is correct](https://bugs.openjdk.org/browse/JDK-8078703)
1. [JDK-8286991: Hotspot container subsystem unaware of VM moving cgroups](https://bugs.openjdk.org/browse/JDK-8286991)
1. [JDK-8286212: Cgroup v1 initialization causes NPE on some systems](https://bugs.openjdk.org/browse/JDK-8286212)
1. [Memory Allocation Table](https://docs.google.com/spreadsheets/d/15OiJ46Xz-Xjdm7brALqHgjdbkLd-C8ghH3bqKMuD9TQ/edit?usp=sharing)
