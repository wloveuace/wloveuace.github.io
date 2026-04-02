
we’ll introduce all major concepts related to memory- both virtual and physical

## ==**History of Virtual memory**==

In old systems and CPUs, Memory was relatively simple, as an application just allocated physical memory directly, used it, freed it, and that was it.

 Each access to memory was a combination of a segment address and an offset, which was needed because these processors worked with 16-bit values internally, but the memory access requires 20 bits (1 MB). A segment register’s value (16-bit) was multiplied by 16 (0x10) and then an offset was added to reach an address in the 1 MB range. This mode of working is now called Real Mode, and is still the mode in which today’s Intel/AMD processors wake up in.
 
 With the introduction of the 80386 processor, virtual memory was born as it’s essentially used today, including the ability to access memory linearly (with the segment registers just set to zero) by using offsets only

>On 64-bit systems, protected mode is called Long Mode,

## ==**What is Virtual memory**==

Virtual memory means every memory access needs to be translated to where the physical address is. This mode is referred to as Protected Mode. In protected mode, there is no way to access physical memory directly- only via a mapping from a virtual address to a physical address. This mapping must be prepared upfront by the operating system’s Memory Manager, since the CPU expects this mapping to be present.

> **Mapping** is the process of associating a virtual memory address with a physical memory address in RAM.

The mapping between virtual and physical addresses, as well as the management of memory blocks on the OS level are performed in chunks called pages. This is necessary, since it’s not possible to manage every single byte.

![[Pasted image 20260402202745.png]]

>Small (normal) pages are the default, and the term “page” used throughout this chapter (and the next ones) means a small or normal page, which is 4 KB on all architectures

## ==**Process Address Space**==

Each process has its own Private, linear, virtual, private address space. The address space starts at address zero and ends at some maximum value, based on the OS

How can multiple processes have same virtual address but different data ?
The answer is **private virtual memory**, For example, stating that there is some data at address 0x100000 requires another question answered: in which process? Every process has an address 0x100000, but that address may be mapped to a different physical address, to a file, or to nothing at all.

![[Pasted image 20260402220840.png]]

A process can directly access memory in its own address space. This means a process cannot accidentally or maliciously read or write to another process’ address space simply by manipulating a pointer. It is possible to access memory of another process, but that requires calling special functions.

>Every process starts with very modest usage of its virtual address space- the executable is mapped as well as `NtDll.Dll`. Then the loader (part of `NtDll`) allocates some basic structures within the process address space, such as the default process heap (discussed in the next chapter), the Process Environment Block (PEB), the Thread Environment Block (TEB) for the first thread in the process. Most of the address space is empty.


## ==**Page states**==

Each page in virtual memory can be in one of three states:
- free 
- committed
- reserved.

**Free pages** are unmapped, and so trying to access a free page causes an access violation exception.

**Committed pages** are the opposite of free pages. A committed page has reserved **backing storage** (either in RAM or on disk, such as a page file or mapped file). This means accessing the page is guaranteed to succeed, but it does **not** mean the page is currently in RAM. Instead, the OS guarantees that the page can be placed in RAM when needed because space has been reserved (in RAM or disk).

If the page is already in RAM (i.e., **resident**), the CPU accesses the data directly and continues execution. If the page is not in RAM, the CPU cannot access it directly (it cannot read from disk), so it raises an exception called a _page fault_. The memory manager handles this by bringing the page into RAM (either by allocating a new frame or loading it from disk), updating the mapping, and then the CPU retries the instruction successfully.

>Committed memory is what normally is called “allocated” memory. Calling C/C++ memory allocation functions, such malloc, calloc, operator new, etc

**Reserved pages** are similar to free, in the sense that accessing that page causes an access violation, but it also ensures that normal memory allocations do not use the range that its been specified because it’s reserved for another purpose.

>We’ve seen this idea in the way a thread’s stack is managed. Since a thread’s stack can grow, and must be contiguous in virtual memory, a range of pages is reserved so that other allocations happening in the process don’t use the reserved address range

## ==**32-bit Systems**==

32-bit means 4 GB, which may have left you wondering why processes get only 2 GBs of virtual address space or 3 GBs MAX ?  The virtual memory is split into 2 , **2 GB user space**, **2 GB kernel space** or 3 : 1. 

![[Pasted image 20260402223455.png]]

>System-reserved virtual memory stores the OS kernel, drivers, memory management structures, and hardware mappings, and is shared across all processes but only accessible in kernel mode.

>Even though System drivers & kernel are singletons but like DLLs, the kernel and drivers exist once in physical memory, and each process’s virtual address space maps to the same physical pages, not copies.

## ==**64-bit Systems**==

The theoretical limit for 64 bits is 2 to the 64ʰ power, or 16 EB,  This is a literally astronomical address range, that seems unreachable in today’s systems. (Limit for 32bit was 4gb).

>Most modern processors support only 48 bits of virtual and physical addresses.

![[Pasted image 20260402225035.png]]

## ==**Address Space Usage**==

To get a sense of what Memory is usable and what's not , We can use the following API

```c
VOID GetNativeSystemInfo(
  [out] LPSYSTEM_INFO lpSystemInfo
);
```
To retrieve accurate information for an application running on WOW64
Parameters:
	- `lpSystemInfo` - A pointer to a [SYSTEM_INFO](https://learn.microsoft.com/en-us/windows/desktop/api/sysinfoapi/ns-sysinfoapi-system_info) structure that receives the information.

![[Pasted image 20260402225653.png]]

>WOW64 is a subsystem that allows 32-bit applications to run on 64-bit Windows.

You can check if a process is WOW64 using:
```c
BOOL IsWow64Process(
  [in]  HANDLE hProcess,
  [out] PBOOL  Wow64Process
);
```
Determines whether the specified process is running under [WOW64](https://learn.microsoft.com/en-us/windows/desktop/WinProg64/running-32-bit-applications) or an Intel64 of x64 processor.
Parameters:
	- `hProcess` - Handle to the process
	- `Wow64Process` - A pointer to a value that is set to TRUE if the process is running under WOW64

>The newer `IsWow64Process2` function provides more information about the processor powering the process and the native processor on the machine. `pProcessMachine` returns one of the `IMAGE_FILE_MACHINE_*` constants defined in `<winnt.h>`


