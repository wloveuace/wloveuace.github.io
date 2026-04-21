---
share_link: https://share.note.sx/29bg8448#9JZq77tnFm6SNTecFZx05nPvEmGzqwzcLxeIQvn3eqw
share_updated: 2026-04-21T02:19:05+02:00
---

In this chapter we will be explaining the process ( The process object, process creation API , process creation under the hood) along with Process jobs.


## ==**Creating a process**==

The Windows API provides several functions for creating processes. The simplest is `CreateProcess`, which attempts to create a process with the same access token as the creating process. If a different token is required, `CreateProcessAsUser` can be used, which accepts an extra argument (the first a handle to a token object that was already somehow obtained

> We can obtain a handle to a Token object by using `LogonUser` function

Other process creation functions include `CreateProcessWithTokenW` and `CreateProcessWithLogonW` (both part of advapi32.Dll) . `CreateProcessWithLogonW` is a handy shortcut to log on with a given user’s credentials and create a process with the obtained token in one stroke

>Secondary Logon service (hosted in a `SvcHost.Exe`) do the actual process creation for `CreateProcessWithTokenW` & `CreateProcessWithLogonW`, SecLogon executes the call the internal `SlrCreateProcessWithLogon` function, and if all goes well, eventually calls `CreateProcessAsUser`.

![[Pasted image 20260419181558.png]]

All the above functions expect a Portable Executable (PE) file , not strictly a .exe, but any PE file (batch file, or 16-bit COM application).

**How are other formats (like .txt) are executed then ?**
The windows shell is the one responsible for such files for example notepad opens with txt files , Functions such as `ShellExecute` and `ShellExecuteEx`. These functions can accept any file (not just executables) and try to locate the executable to run based on the file extensions and the registry settings at `HKEY_CLASSES_ ROOT`

>`ShellExecute(Ex)` calls `CreateProcess` with a proper executable and appends appropriate arguments on the command line to achieve the user’s intention

#### CreateProcess Arguments

Its worth explaining what `CreateProcess` needs to know before making a process. 
1. The executable path and command-line arguments.
2. Optional security attributes applied to the new process and thread object that’s about to be created
3. A Boolean flag indicating whether all handles in the current (creating) process that are marked inheritable should be inherited (copied) to the new process.
4. Various flags that affect process creation.
5. An optional environment block for the new process (specifying environment variables).
6. An optional current directory for the new process. (If not specified, it uses the one from the creating process.)
7. A STARTUPINFO or STARTUPINFOEX structure that provides more configuration for process creation.
8. A PROCESS_INFORMATION structure that is the output of a successful process creation.

*Explained in details in windows system programming*

## ==**Process internals**==

Each Windows process is represented by an executive process (EPROCESS) structure. an EPROCESS contains and points to a number of other related data structures. For example, each process has one or more threads, each represented by an executive thread (ETHREAD) structure.

The EPROCESS is found in kernel address space , but Some of its data structures are accessed by User space because they contain information needed by the code , One important example is **Process Environment Block (PEB)**, Additionally, some of the process data structures used in memory management, such as the **working set list**

Beside the kernel , other system services create their own structures for each process that is executing, for example the Windows subsystem process (Csrss) maintains a parallel structure called the `CSR_PROCESS` , A struct called `W32PROCESS` which is created the first time a thread calls a Windows USER or GDI function , A struct called `DXGPROCESS` for the Graphics Device Interface (GDI) component

>Many other drivers and system components, by registering process-creation notifications, can choose to create their own data structures to track information they store on a per-process basis.

EPROCESS structure:
![[Pasted image 20260419203417.png]]

the first member of the executive process structure is called PCB (Process Control Block). It is a structure of type KPROCESS, for kernel process. the dispatcher, scheduler, and interrupt/time being part of the operating system kernel use the KPROCESS instead of EPROCESS

KPROCESS structure:
![[Pasted image 20260419203729.png]]

The data structure representing a process's information for user mode space is called PEB, It contains information needed by the image loader, the heap manager, and other Windows components that need to access it from user mode 

Process Environmental Block structure:
![[Pasted image 20260419204553.png]]

Other process structures ( `CSR_PROCESS` , `W32PROCESS` ):
![[Pasted image 20260420004915.png]]
![[Pasted image 20260420004918.png]]


## ==**Flow of create process**==

we’ll see how and when the process data structures are created and filled out, as well as the overall creation and termination behaviors behind processes.

Because of the multiple-environment creating an executive process object consists of 3 system parts:
1. The windows client library `kernel32.dll`
2. The windows executive
3. Windows subsystem process (Csrss)

The following list summarizes the main stages of creating a process:
- Validate parameters; convert Windows subsystem flags , and attributes to NT_dll form
- Open the image file of the process
- Create a windows executive process object
- Create the initial thread (stack, context, and Windows executive thread object).
- Start execution of the initial thread (unless the CREATE_SUSPENDED flag was specified)
- Loader then loads required dlls and the main thread runs

### Step 1 (CreateProcessInternalW)
Before opening the executable image to run, `CreateProcessInternalW` performs the following steps:
-  assign priority class to the process by choosing the lowest-priority class set.
- If the creation flags specify that the process will be debugged, Kernel32 initiates a connection to the native debugging code in Ntdll.dll and gets a handle to the debug object from the current thread’s environment block (TEB).
- The user-specified attribute list is converted from Windows subsystem format to native format
- The security attributes for the process and initial thread are converted to their internal representation
- The application and command-line arguments passed are analyzed
- Most of the gathered information is converted to a single large structure of type `RTL_USER_ PROCESS_PARAMETERS` (Pointed to by the process’s PEB (Process Environment Block)

![[Pasted image 20260420014001.png]]

Once these steps are completed, `CreateProcessInternalW` performs the initial call to `NtCreateUserProcess` to attempt creation of the process.

### Step 2 (Open the image)

At this point, the creating thread has switched into kernel mode and continues the work
- Revalidate the arguments and build an internal structure to hold all creation information
- find the appropriate Windows image that will run the executable file specified by the caller and  create a section object to later map it into the address space of the new process.
- create a section object for the executable , Just because a section object has been successfully created doesn’t mean the file is a valid Windows image as it could be a DLL or other image
- looks in the registry under the Image file execution options (IFEO) located in `HKLM\ SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options` to see whether a subkey with the file name and extension of the executable image exists

![[Pasted image 20260420014952.png]]


## Step 3 (Create the process object)

At this point, `NtCreateUserProcess` has opened a valid Windows executable file and created a section object to map it into the new process address space. Next, it creates a Windows executive process object to run the image

Creating the executive process object involves the following:
- *A* Setting up the `EPROCESS`object
- *B* Creating the initial process address space
- *C* Initializing the kernel process structure (KPROCESS)
- *D* Concluding the setup of the process address space
- *E* Setting up the PEB

### A - Setting up the EPROCESS object

This is the longest process since its the biggest structure and it contains all the process's information *Security*, *Lookup-information*, *Affinity*, *Priority and performance*

1. Inherit the affinity of the parent process unless it was explicitly set during process creation (Affinity allows  a  process  or  thread to run on specific CPU cores to optimize performance and cache usage. )
2. Choose the ideal NUMA node that was specified in the attribute list, if any.
	>**NUMA (Non-Uniform Memory Access)** is a hardware architecture where a system is divided into "nodes." Each node consists of a group of processors and their own dedicated local memory. (- **Local Access:** A CPU accessing memory within its own node is very fast , **Remote Access**: accessing memory within another node and slower)
3. Inherit the I/O and page priority from the parent process. (Normal is 5) - All pages get same priority unless dynamically changed
4. Set the new process exit status to STATUS_PENDING.
5. Store the parent process’s ID in the `InheritedFromUniqueProcessId` field in the new process object
6. Query the Image File Execution Options (IFEO) key to check if the process should be mapped with large pages, if WOW64 then it wont have large pages
7. Query the performance options key in IFEO (`PerfOptions`, if it exists), which may consist of any number of the following possible values: `IoPriority`, `PagePriority`, `CpuPriorityClass`, and `WorkingSetLimitInKB`.
>The **Image File Execution Options (IFEO)** are located in the Windows registry at the following path: `HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Image File Execution Options`
8. Create the process’s primary access token (a duplicate of its parent’s primary token). Or another token if specified
9. The process minimum and maximum working set sizes are set to the values of `PspMinimumWorkingSet` and `PspMaximumWorkingSet`. 
   >These values can be overridden if performance options were specified in the `PerfOptions` key part of Image File Execution Options
10. Initialize the address space of the process.
11. Initialize the KPROCESS part of the process object.
12. The process’s priority class is set to normal (If parent had below normal , same class priority would be inherited)
13. The process handle table is initialized. If the inherit handles flag is set for the parent process, any inheritable handles are copied from the parent’s object handle table into the new process.
14. The final process priority class and the default quantum for its threads are computed and set
15. The various mitigation options provided in the IFEO key (as a single 64-bit value named Mitigation) are read and set.

Summarized Process:
- Inherit I/O pages priority & affinity
- Store parent process id
- Check IFEO for performance option keys (priorates) and Working set flags
- Create process token and assign it
- Initialize process address space
- Set process priority class
- Create the process Handle and inherit parent objects
- Check for mitigation options in IFEO

### **B - Creating the initial process address space**

The initial process address space consists of the following pages:
- Page directories which contain page tables
- Hyperspace page
- VAD bitmap page
- Working set list

To create these pages, the following steps are taken:
1. Page table entries are created in the appropriate page tables to map the initial pages.
2. The number of pages is deducted from the kernel variable `MmTotalCommittedPages` and added to `MmProcessCommit`. (Committed since its about to be written to)
3. The system-wide default process minimum working set size (`PsMinimumWorkingSet`) is deducted from `MmResidentAvailablePages`.
4. The page table pages for the global system space are created

Summarized Process:
- Create page table entries and map them to initial pages
- Number of pages available for system is updated
- The working set size is updated

### **C - Creating the kernel process structure**

The next stage is the initialization of the KPROCESS structure. Most of the info initialized and created here are towards the process threads since the scheduler and dispatcher are interested in those info

1. The doubly linked list, which connects all threads part of the process is initialized
2. The initial value (or reset value) of the process default quantum is set to 6
3. The process’s base priority is set
4. The default processor affinity for the threads in the process is set

### **D - Concluding the setup of the process address space**

*To be continued*

## ==**Protected process**==

In windows , Normal users can have privileges on processes that gives them access to Close , inject code , and suspend threads in that process

This logical behavior (which helps ensure that users will always have full control of the running code on the system) clashes with the system behavior for digital rights management requirements imposed by the media industry

Windows solved this issue by introducing Protected processes. The Protected Media Path (PMP) in Windows makes use of protected processes to provide protection for high-value media, and developers of applications such as DVD players can make use of protected processes by using the Media Foundation (MF) API.

Protected processes can be created by any application. However, the operating system will allow a process to be protected only if the image file has been digitally signed with a special Windows Media Certificate.

Windows adds 2 changes in the protected process creation:
1. process creation occurs in kernel mode to avoid injection attacks
2. protected processes have special bits set in their EPROCESS structure that modify the behavior of security-related routines in the process manager to deny certain access rights that would normally be granted

>the only access rights that are granted for protected processes are PROCESS_QUERY/SET_LIMITED_INFORMATION, `PROCESS_TERMINATE` and `PROCESS_SUSPEND_RESUME`.

>Attempt by a driver to change the EPROCESS flag indicating the Protected process will lead to a violation of the PMP model and be considered malicious, and such a driver would likely eventually be blocked from loading


*To be continued*