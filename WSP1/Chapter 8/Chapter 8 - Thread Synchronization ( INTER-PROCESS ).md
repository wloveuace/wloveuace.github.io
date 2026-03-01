---
share_link: https://share.note.sx/eqjnk8sw#65CArxS4SHNRqGhO1sm31Xg6Uyw7wrV/ZsVe3KWQGtU
share_updated: 2026-02-10T03:17:19+02:00
---


### ==**Dispatcher Objects**==

this chapter talks about INTER PROCESS SYNC, which means synchronization between threads of different processes, but before talking about new kernel objects used in the process , we gonna do a quick revision about Kernel objects:

- Kernel objects reside in system (kernel) space and are theoretically accessible from any process, provided that process can obtain a handle to the requested object
- Handles are process relative
- There are three ways to share objects across processes: handle inheritance, name, and handle duplication

Some kernel objects are more specialized, called dispatcher objects or waitable objects. Such objects can be in one of two states: signaled or non-signaled. The meaning of signaled and non-signaled depends on the type of object

`Dispatcher objects - diagram 1`
![[Pasted image 20260210013836.png]]

```
WaitForSingleObject() - wait for a single object to be signaled[^1]
WaitForMultipleObjects - waits for multiple objects to be signaled ( max of 64 objects at a time )[^2]
```

`Wait return values - diagram 2`
![[Pasted image 20260210014652.png]]

What happens if multiple threads wait for the same mutex, and it becomes signaled? Only one of the threads can acquire the mutex before it flips back to the non-signaled state. Behind the scenes waiting threads for an object are stored in a first-in-first-out (FIFO) queue, so the first thread in the queue is the one woken up (regardless of its priority)

### ==**The Mutex**==

The mutex (short for “mutual exclusion”) provides similar functionality to the critical section. Its purpose is the same: protect shared data from concurrent access. Only one thread at a time can acquire the mutex successfully, and proceed to access the shared data. All other threads waiting for the mutex must continue to wait until the mutex is released by the acquiring thread

```
CreateMutex()/Ex - create a mutex object with a name
OpenMutex() - opens a mutex handle with name
```

acquiring a mutex can be done with the wait functions and when its signaled ownership is passed to the waiting thread automatically

```
ReleaseMutex() - release mutex 
```

What happens if a thread that owns a mutex exits or terminates (for whatever reason)? Since the owner of the mutex is the only one that can release the mutex, this may cause a deadlock, where other threads waiting for the mutex will never acquire it. This kind of mutex is called abandoned mutex, literally abandoned by its owner thread

### ==**The Semaphore**==

The purpose of a semaphore is to limit something, in a thread-safe way. A semaphore is initialized with a current and a maximum count. As long as its current count is above zero, it’s in the signaled state. Whenever a thread calls `WaitForSingleObject` on a semaphore and it’s in the signaled state, the semaphore’s count is decremented and the thread is allowed to proceed.

```
CreateSemaphore() - creates a semaphore object with a name
OpenSemaphore()
ReleaseSemaphore() - release a semaphore and incremnt semaphore counter *The function allows specifying the count to release (i.e. how many to add to the current semaphore’s count). This value is typically 1 but can be higher.*
```


### ==**The Event**==

the event is just a flag that can be set (signaled state) or reset (non-signaled). Being a (possibly named) kernel object gives it the flexibility to work within a single process or across processes.

Events have 2 modes:
- Manual reset: puts the event in the signaled state, and releases all threads waiting on it (if any). The event remains in the signaled state
- Auto reset: a single thread is released from wait, and then the event goes back automatically to the non-signaled state

```
CreateEvent() - creates an event object with a name
OpenEvent()

SetEvent() - sets event to signaled
ResetEvent() - sets event to non-signaled
PulseEvent() - set the event momentarily and if no thread is currently waiting, reset the event
```

### ==**The waitable timer**==

A waitable timer becomes signaled when its due time arrives and executes a callback function.

```
CreateWaitableTimer() - create a waitable timer with name
OpenWaitableTimer() 
```

callback function:
```
typedef VOID (CALLBACK *PTIMERAPCROUTINE)( 
 _In_opt_ LPVOID lpArgToCompletionRoutine,
 _In_ DWORD dwTimerLowValue,
 _In_ DWORD dwTimerHighValue
);
```

```
SetWaitableTimer() - set waitiable timer object execution time and a callback [^3]
```

Timer can have 2 modes:
- Periodic designed to activate code through an APC, which can be challenging due to re-entrancy issues. They are suitable for scenarios where a thread is otherwise occupied but blocks often enough to allow an APC to run.
- One-shot: an be used when a single execution is sufficient. They are less complex and do not  require the thread to be blocked for an extended period.

the callback function is wrapped in an Asynchronous Procedure Call (APC), APCs are pushed at the end of the particular thread’s queue, but in order to run them the thread must enter an alertable state. In this state, the thread first checks if any APCs have accumulated in its APC queue, and if so, runs all of them now in sequence before resuming execution of the code following its entering the alertable state.

```
SleepEx() - puts the thread in alertable state [^4]
```


Another function worth mentioning is the `timeSetEvent` function, which executes your callback on a **system-managed timer thread**, not your main thread. which is:
- A thread created internally by the multimedia timer subsystem
- Shared between timers in the process
- Not directly controllable by you
- thread runs in THREAD_PRIORITY_TIME_CRITICAL (+15)

```
timeSetEvent() - set timer event with call back run by system-managed timer thread
```

## Functions discussed in this chapter

```
WaitForMultipleObjects()
WaitForSingleObject()

CreateMutex()
OpenMutex()
ReleaseMutex() 

CreateSemaphore()
OpenSemaphore()
ReleaseSemaphore()

CreateEvent() 
OpenEvent()
SetEvent()
ResetEvent()
PulseEvent()

CreateWaitableTimer()
OpenWaitableTimer()
SetWaitableTimer()
timeSetEvent()

SleepEx()

SystemTimeToFileTime()
```

---

[^1]: setting the timeout to 0 will just check if the object is signaled or not without waiting

[^2]: return value is between WAIT_OBJECT_0 and WAIT_OBJECT_0+count-1, where count is the number of handles.

[^3]: timer is in FILETIME structure so we can use `SystemTimeToFileTime` to change a SYSTEMTIME to a FILETIME, The FILETIME structure is identical to a LARGE_INTEGER, which is a 64bit int (2 ints LOW & HIGH)

[^4]: Calling SleepEx with bAletrable set to TRUE puts the thread to sleep in an alertable state
