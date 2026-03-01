---
share_link: https://share.note.sx/bew42hht#8AXGmJ6/XgYyRywRD+qp7RRChyqJ0lFlvS9d2oiYsH8
share_updated: 2026-02-08T17:08:42+02:00
---
### ==**Priorities**== 

Threads are meant to execute, and from previous chapter we know that number of threads that run at once is dependent on the number of logical processors, so what if there is a process that has to do critical work , and there are multiple threads executing from other programs, how will the OS handle such issue and how much time does a thread take to execute, how much time will it wait etc

Priorates make make critical threads skip and pass the queue of threads that can wait, and this is done through priority.

Thread priorities are from 0 to 31, where 31 is the highest. Technically, thread 0 is reserved for a special thread called zero page thread that is part of the memory manager in the kernel. It’s the only thread that is allowed to have a priority of zero. So technically, usable priorities are from 1 to 31. 

the priorities cannot be set to an arbitrary value. Instead, a thread’s priority is a combination of a process’ Priority Class (called Base Priority in Task Manager) and an offset around that base priority

`Process priorities - diagram 1`
![[WSP1/Chapter 6/d1.png]]

The values in (diagram 1) are the Process priority class values, you will be asking well where is for-example the 31th priority, that is where the Thread priority comes in and adds more value to the priority rank [^1]

```
SetPriorityClass() - used to set process priority class [^2]
GetPriorityClass() - get priority class of a process
```

The process priority class has no direct effect on the process itself, since a process does not run- threads do. All threads created in a process have their default priority set to the priority class level, so the threads priorities will change not the process itself. ( sort of a method to change the priority to all threads at once ) 

`Thread Priorites - diagram 2`
![[WSP1/Chapter 6/d2.png]]

```
SetThreadPriority() - change the priority for a thread
```

**Relative thread priority**: the priority of a thread + the priority of the process
**Base priority:** the priority set by the developer
**Dynamic priority:** is the actual current priority for that thread. (In some cases the priority is increased (boosted) temporarily)

`Relative thread priorities - diagram 3`
![[WSP1/Chapter 6/d3.png]]
[^3]

### ==**Scheduling Basics**== 

==**Quantum**== Amount of time a CPU gives to a thread to execute, The default quantum is 2 clock ticks for client machines and 12 on server machines[^5][^6]

==**Single CPU Scheduling**== The scheduler maintains a Ready Queue, where threads that want to execute (are in the Ready state) are managed. All other threads that do not want to execute at this time (in the Wait state) are not being looked at since they don’t want to execute

`Single CPU Scheduling - diagram 4 `
![[WSP1/Chapter 6/d4.png]]

The algorithm for a single CPU goes like this:
1. The highest priority thread runs first
2. when its quantum expires, the scheduler preempts thread 1, saving its state in its kernel stack, and it goes back to the Ready state (since it still has things to do). Thread 2 now becomes the running thread since it has the same priority
3. Threads execute from highest priority threads to low priority threads
4. windows gives low priority threads a thread boost to priority 15 every roughly 4 seconds to have a chance to execute and the priority decreases gradually to the original priority every quantum[^4]
5. If a high priority thread was in a wait state and a low priority thread was running, the os interrupts the low thread quantum and lets the high thread do its work and low thread back to ready state

==**Thread wait states**==
• Performing a synchronous I/O operation 
• Waiting on a kernel object, that is currently not signaled
• Waiting for a UI message when there are none
• Entering a voluntary sleep


---
[^1]: The priority class of a process can be set when that process is being created with `CreateProcess`. The sixth parameter (flags) can be combined with one of the constants in diagram 1

[^2]: if the target priority class is Real-time, the caller must have the `SeIncreaseBasePriority` privilege

[^3]: From the kernel’s scheduler perspective, only the final number is important. It doesn’t care how that number came to be. (the dynamic priority is the determining priority value)

[^4]: this idea is called CPU starvation (when a thread low in priority doesnt get CPU time)

[^5]: There are other ways to change the quantum. For fine-grained control, the registry value HKLM\SYSTEM\CurrentControlSet\Control\PriorityControl\Win32PrioritySeparation controls not just the length of a quantum, but also quantum stretching for the foreground process

[^6]: Another way to control the quantum is available by using a job object (described in detail in chapter 4) with the Scheduling Class field in JOBOBJECT_BASIC_LIMIT_INFORMATION.
