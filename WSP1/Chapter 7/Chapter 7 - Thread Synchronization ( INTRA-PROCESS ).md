---
share_link: https://share.note.sx/xciru53i#z5ml9tDc8uiHUdoovXXbzgm60Jgs7Yhn7y9tuhDld/Y
share_updated: 2026-02-10T16:36:42+02:00
---
# ==**Synchronization Basics**== 

synchronization is all about data races, data race is then 2 threads are writing so a specific memory address at the same time, reading is never an issue but writing is where complications happen

Most synchronization-related operations require threads to wait upon some condition, until it is safe to proceed, preventing a data race.

### ==**Atomic operations**==

This atomic operation and other similar operations are exposed in the Windows API though a set of functions with the Interlocked prefix. The Interlocked family of functions is great with simple cases such as integer increments. In the simple increment case, that’ `InterlockedIncrement`:

```
unsigned InterlockedIncrement(unsigned volatile *Addend); - This performs an atomic increment, and as a bonus, returns the new value in addition to actually changing the memory location [^1]
```
[^2]

atomics shall not be used on more heavier cases

### ==**Critical Sections**==

A critical section is a classic synchronization mechanism based on one thread at most acquiring a lock. Once a thread acquired a specific lock, no other thread can acquire the same lock until the thread that acquired it in the first place releases it. Only then, one (and only one) of the waiting threads can acquire the lock. This means that at any given moment, no more than one thread has acquired the lock.

`Critical section workflow - diagram 1`
![[Pasted image 20260207200224.png]]

The thread that acquired the lock is also its owner, which means two things:
- The owner thread is the only thread that can release the critical section
- If the owner thread attempts to acquire the critical section a second time (recursively), it succeeds automatically, incrementing an internal counter. This means the owner thread now has to release the critical section the same number of times to truly release it.

```
InitializeCriticalSection()/Ex - used to create a critical section object
InitializeCriticalSectionAndSpinCount() - makes waiting thread to spin lock on the section object [^3]

EnterCriticalSection() - attempts to acquire the critical section and only returns when it does.[^4]
LeaveCriticalSection() - releases an already acquired critical section

DeleteCriticalSection() - delete critical section object
```

**The code between acquire and release of the lock is called a critical region.**

The `EnterCriticalSection` waits for the critical section to be available as long as it takes. There is no way to specify a timeout, but there is a way to inspect the critical section, if it’s free- acquire it; otherwise, continue execution.

```
TryEnterCriticalSection() - inspect the critical section, if it’s free- acquire it; otherwise, continue execution.
```

As we’ve seen in the previous section, `EnterCriticalSection` and `LeaveCriticalSection` are natural pairs. It would be unfortunate to “forget” calling `LeaveCriticalSection`, for example by returning from the function before the call. This mistake is easy to make, and even if there is no such bug, it forces the developer to think about it and make sure that any future modifications to that function don’t break the pair of calls

It would be much better to have code that automatically calls `LeaveCriticalSection` no matter what, without the code having to worry about it. There are two ways to get this behavior: termination handlers and C++ Resource Acquisition Is Initialization (RAII)

==**termination handlers**==

```c
#include <stdio.h>
#include <Windows.h>

CRITICAL_SECTION criticalSection;

int main() {

	InitializeCriticalSection(&criticalSection);

	__try {
		EnterCriticalSection(&criticalSection);
		//Do work
	}
	__finally {
		LeaveCriticalSection(&criticalSection);
	}
}
```

The \__try and \__finally are two Microsoft-specific keywords extending the C language for the sake of running the code in the \__finally block when leaving the \__try block, no matter what. Even a return statement within the \__try block would first call the \__finally block and only then actually return from the function.[^5]

==**Deadlocks**==
Working with critical sections seems simple enough. Even if we work with the various RAII wrappers, there is still a danger of deadlocks. A classic deadlock occurs when thread A that owns lock 1 (e.g. a critical section) waits for lock 2 that is owned by thread B, while thread B is waiting for lock 1. The way to avoid deadlocks is theoretically easy: always acquire the locks in the same order. This means that every thread that needs more than one lock should always acquire the locks in the same order. This guarantees deadlock cannot happen

### ==**Reader Writer Locks**==

Using critical sections to protect shared data from concurrent access works well, but it’s a pessimistic mechanism- it allows one thread at most to access the shared data. In some scenarios, where some threads read the data, and other threads write the data, an optimization can be made: If one thread reads the data, there is no reason to prevent other threads that only read the data from doing it concurrently. This is exactly the role of the “Single Writer Multiple Readers” mechanism

```
InitializeSRWLock() - initializes a SRWLOCK object[^6]
```

SRW locks have 2 modes ( shared and exclusive ), Shared mode allows concurrent read access by multiple threads, while exclusive mode grants access to only one thread for writing. So in exclusive mode , there are no readers just 1 thread writer [^7]

```
AcquireSRWLockShared()
AcquireSRWLockExclusive()

ReleaseSRWLockShared()
ReleaseSRWLockExclusive()

TryAcquireSRWLockExclusive()
TryAcquireSRWLockShared()
```

### ==**Condition Variables**==

Condition variables are another synchronization mechanism, providing the capability to wait on a critical section or SRW lock until some condition occurs. A classic example of using condition variables is a producer/consumer scenario. Suppose some threads produce data items and place them in a queue. Each thread does whatever work is needed to produce the items. At the same time, other threads act as consumers- each removing an item from the queue and processes it in some way

If items are produced faster than consumers can process, then the queue is non-empty, and consumers continue working. On the other hand, if consumer threads process all items, they should go into a wait state until new items are produced, in which case they should be awoken. This is exactly the behavior provided by condition variables.

`Produces and consumer threads - diagram 2`
![[Pasted image 20260208054153.png]]

```
InitializeConditionVariable() - initialize a conditional variable object
```

A condition variable is always associated with a critical section or SRW lock. When a thread needs to wait until a condition variable is signaled, it first must acquire the critical section/SRW lock and then call the associated sleep function

```
SleepConditionVariableCS() - wait till condition is singaled and locks critical section (if not signaled yet , the section isnt locked )
SleepConditionVariableSRW() - wait till condition is singaled and locks SRW(if not signaled yet , the section isnt locked )
```


The function releases the synchronization object and waits on the condition variable, atomically. While waiting, the thread may be woken by calling one of the wake functions on the condition variable

```
WakeConditionVariable()
WakeAllConditionVariable()
```

`WakeConditionVariable` wakes a single thread (no guarantee on which thread it is if multiple threads are sleeping on the condition variable), while `WakeAllConditionVariable` wakes all threads waiting on the condition variable

`consumer thread workflow - diagram 3`
![[Pasted image 20260208054818.png]]

>TLDR;
>Condition variables work with locks to coordinate producer-consumer scenarios by allowing threads to wait efficiently when the queue is empty (consumers) or full (producers). They provide a mechanism to signal waiting threads only when their conditions are met, preventing busy-waiting and race conditions.
>![[Pasted image 20260208055305.png]]

[^8]
Demo of using conditional variables [Code](https://github.com/kontaie/Consumer-Producer-thread-demo/tree/main)

### ==**Waiting on Address**==

Windows 8 and Server 2012 adds another synchronization mechanism, that allows a thread to wait efficiently until the value at some address changes to a desired value. Then it can wake up and proceed with its work. It’s certainly possible to use other synchronization mechanisms to achieve a similar effect, such as using a condition variable, but waiting on address is more efficient and is not prone to deadlocks since no critical sections (or other software synchronization primitives) are used directly.

```
WaitOnAddress() - A thread can enter a wait state until a certain value appears on a “monitored”[^9]
```

Some other thread may change the value in *Address. Unfortunately, this does not automatically cause the waiting thread to wake. Instead, the thread that made the change must call one of the “wake” functions

```
WakeByAddressSingle()
WakeByAddressAll()
```

### ==**Synchronization Barriers**==

Another object introduces in windows 8 is Synchronization Barriers, They're used to ensure multiple threads reach a certain point in execution before any of them can proceed further

```
InitializeSynchronizationBarrier() - initialized sync barrier object
EnterSynchronizationBarrier() - stops thread execution and wait till other threads finish their work and stop at the same position
DeleteSynchronizationBarrier()
```

### Functions discussed in this chapter:
```
Interlocked functions - /*this is just a pattern

InitializeCriticalSection()/Ex
InitializeCriticalSectionAndSpinCount()

EnterCriticalSection()
TryEnterCriticalSection()
LeaveCriticalSection()
DeleteCriticalSection()

InitializeSRWLock()
AcquireSRWLockShared()
AcquireSRWLockExclusive()

TryAcquireSRWLockExclusive()
TryAcquireSRWLockShared()

ReleaseSRWLockShared()
ReleaseSRWLockExclusive()

InitializeConditionVariable()
SleepConditionVariableCS()
SleepConditionVariableSRW()

WakeConditionVariable()
WakeAllConditionVariable()

WaitOnAddress()
WakeByAddressSingle()
WakeByAddressAll()

InitializeSynchronizationBarrier()
EnterSynchronizationBarrier()
DeleteSynchronizationBarrier()
```

---

[^1]: Behind the covers, this is not a true function, but rather a compiler intrinsic that issues a special instruction to the CPU to perform this operation atomically. This is great, since leveraging the hardware is always going to be faster than software. Also, since there is no explicit “lock” object used, no deadlock is possible with these functions.

[^2]: author of this book has made a great video about concurrency [video]([Concurrency and the C++ Memory Model](https://www.youtube.com/watch?v=NZ_ncor_Lj0))

[^3]: A compromise could be to spin a small amount of time because it’s likely that the current owner of the critical section will release it very soon, and so a transition to the kernel can be avoided. The default spin count is 2000 (used by `InitializeCriticalSection`)

[^4]: Each call to `EnterCriticalSection` must be matched by a `LeaveCriticalSection` in the same function. It’s too dangerous to call some other function within the critical region that is expected to call `LeaveCriticalSection`

[^5]: Internally, Microsoft's try/finally uses SEH (Structured Exception Handling) which manipulates the exception handler chain. It creates assembly-level EXCEPTION_REGISTRATION records on the stack and registers them with the OS via fs:\[0] register (x86) or gs:\[0] (x64). The compiler transforms these blocks into state machine code that tracks execution flow.

[^6]: Alternatively, the structure can be initialized statically by assigning it to SRWLOCK_INIT macro, which just zeros out the structure. Curiously enough, there is no “delete” for the SRWLOCK; this is because all of its internal information is packed into that pointer-sized cell.

[^7]: An exclusive owner cannot recursively acquire a lock; this causes a deadlock. There is no guarantee that the first thread acquiring a lock is the first to receive it. As the documentation states: “SRW locks are neither fair nor FIFO”.

[^8]: condition variables can wake up spuriously ( without being signaled) so u must always recheck the condition in a while loop like the `while` loop rechecks your condition to make sure you should actually proceed without it, your code thread thing could proceed n wake up when the queue is still empty and crash

[^9]: The values size to compare is specified in the AddressSize parameter, and it must be 1, 2, 4 or 8.
