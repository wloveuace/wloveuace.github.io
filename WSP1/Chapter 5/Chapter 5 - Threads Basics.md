---
share_link: https://share.note.sx/kd1k1rte#CvCoI1czh/jSG2fvUUdDN96g7dbDzfPhYt0WzeKgFk8
share_updated: 2026-02-08T17:11:15+02:00
---
# ==**Threads**== 

Threads are the executive part of a process, that runs all the code and uses system resources. Each thread is an independent path of execution ,and so can use a different processor, resulting in true concurrency

### ==**Sockets, Cores and Logical Processors**==

Threads represent multiple processors, in modern CPU days , In the days where multiple cores make up a typical CPU 

==**Sockets**== which is the physical chip stuck in the computer’s motherboard and are the physical connections between CPU cores . Laptops and home computers typically have just one of these.

==**Core**== Contains more than one Logical processor

==**Logical Processor**== also called Hardware threads, From Windows perspective, the number of processors is the number of logical processors (diagram 1). This means that at any given moment, at most 16 threads may be running[^1]

`Typical modern cpu - diagram 1`
![[WSP1/Chapter 5/d1.png]]

### ==**Creating and Managing Threads**== 

```
CreateThread() - function used to create a thread, threads must follow the[^4] following digram: DWORD WINAPI ThreadProc(_In_ PVOID pParameter);
GetExitCodeThread() - Get thread exit code
ResumeThread() - run suspended thread
OpenThread() - obtain thread handle
```
[^2]

once a thread is created, we wait for it to finish execution and return its exit code with the function:
```
WaitForSingleObject() - wait for a single object to finish execution ( objects include threads )
WaitForMultipleObjects() - wait for a multiple object to finish execution
GetThreadTimes() - The function returns a thread’s creation time, exit time, the time it spent executing in kernel mode and the time it spend executing in user mode[^3]
``` 

since more and more threads will cause context switches to occur, because not all threads can execute at the same time, its best to use max number of threads as that of number of logical processors which is critical in apps that need performance

Terminating a thread can be done with the following options:
1. The thread function returns (best option) 
2. The thread calls `ExitThread` (best to avoid) 
3. The thread is terminated with `TerminateThread` (typically a bad idea)

### ==**Thread’s Stack**==

Local variables and return addresses from functions reside on a thread’s stack. The size of a thread’s stack can be specified with the second parameter to `CreateThread`, but there are actually two values that affect a thread’s stack: a reserved memory, committed memory

Reserved memory: Memory saved for the thread stack and cant be used by any other thread (not-mapped)
committed: Memory that is allocated and marked as Ready to use memory (mapped)

It’s possible to allocate the maximum stack size immediately, committing the entire stack upfront, but that would be wasteful, as a thread might not need the entire range for its stack related work.

The memory manager has an optimization up its sleeve: commit a smaller amount of memory and if the stack grows beyond that amount, trigger an expansion of the stack, up to the reserved limit. The triggering is done by a page with a special flag, PAGE_GUARD that causes an exception if touched. This exception is caught by the memory manager, which then commits an additional page, moving the PAGE_GUARD page one page down(remember that a stack grows to lower addresses).[^5]

Stack size is some what configurable , ranging from 1mb to 8mb

The default commit size for a thread’s stack in Notepad is (68 KB) and the reserved size is (512 KB).

```
SetThreadStackGuarantee() - attempt at guaranteeing a certain stack size
SetThreadDescription() - sets a name for the thread
GetThreadDescription() - get thread name
```

`Thread stack in action - diagram 2`
![[WSP1/Chapter 5/d2.png]]


#### Functions discussed in this chapter 
```
CreateThread()
OpenThread()

GetThreadTimes()
FileTimeToSystemTime()

SuspendThread()
ResumeThread()

ExitThread()
TerminateThread()
GetExitCodeThread()

WaitForSingleObject()
WaitForMultipleObjects()

SetThreadStackGuarantee()
SetThreadDescription()
GetThreadDescription()
```

---

[^1]: Number of logical processors represent the number of threads that a CPU can run at once and this technology is called HYPER-THREADING

[^2]: `dwCreationFlags` Changes the initial state of a thread( suspended, running )

[^3]: The kernel and user times are reported in a FILETIME structure ,which is a 64-bit value stored in two 32-bit values
	```
	typedef struct _FILETIME { 
		 DWORD dwLowDateTime;
		 DWORD dwHighDateTime; 
	} FILETIME, *PFILETIME, *LPFILETIME;
	```
	The value is in 100 nanosecond units (10 to the-7th power), which means the value in milliseconds can be obtained by dividing by 10000.

[^4]: The thread in fact starts execution inside an NTDLL.dll function named `RtlUserThreadStart`, which conceptually calls the thread’s actual function as provided to `CreateThread`.

[^5]: The actual minimum for a guard page is 12 KB, meaning 3 pages. This guarantees that a stack expansion will allow at least 12 KB of committed memory to be available for the stack.
