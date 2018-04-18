## Overview

> The term *work conserving* is used to describe a scheduler that permits the CPU to run at 100% if any virtual machine (or process, in the case of operating schedulers) has work to do. A non-work-conserving scheduler imposes a hard limit on the amount of CPU time a given process can consume. Work conserving schedulers are preferred in a lot of cases, because they allow the most efficient usage of the CPU. In some situations, it is desirable to limit the total amount of CPU time a VM can consume - for example, to conserve power or for billing purposes. In these cases, a non-work-conserving scheduler is desirable.

## VM Scheduling Parameters
Each domain (including Domain0) is assigned a **weight** and a **cap**.

### Weight
A domain with a weight of 512 will get twice as much CPU as a domain with a weight of 256 on a contended host.

### Cap
The cap, if set, fixes the maximum amount of CPU a domain will be able to consume, even if the host has idle CPU cycles.

## Technical Details
### Algorithm
Each physical CPU manages a local run queue of runnable virtual CPUs. This queue is sorted by vCPU priority. A vCPU's priority can be one of two value: `OVER` or `UNDER` representing wether this vCPU has or hasn't yet exceeded its fair share of CPU resource in the ongoing accounting period. When inserting a vCPU in a run queue, it is put after all other vCPUs of the same priority.

>  At the beginning of an accounting period, each domain is given **credit** according to its **weight**, and the domain distributes the credit to its VCPUs.

As a VCPU runs, it consumes **credits**. Every so often, a system-wide accounting thread recomputes how many credits each active VM has earned and bumps the credits. Negative credits imply a priority of `OVER`. Until a vCPU consumes its alloted credits, it priority is `UNDER`.

On each CPU, at every scheduling decision (when a vCPU blocks, yields, completes its time slice, or is awaken), the next vCPU to run is picked off the head of the run queue.

The Credit scheduler uses **30ms** time slices for CPU allocation. A VM (VCPU) receives 30 ms before being preempted to run another VM. Once every 30ms, the priorities (credits) of all runnable VMs are recalculated.

### SMP load balancing
When a CPU doesn't find a vCPU of priority `UNDER` on its local run queue, it will look on other CPUs for one. This helps making sure that each VM receives its fair share of CPU time. Before a CPU goes idle, it will look on other CPUs to find any runnable vCPU. This guarantees that the scheduler act as a work-conserving one.
