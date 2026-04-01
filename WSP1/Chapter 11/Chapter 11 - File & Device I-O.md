---
share_link: https://share.note.sx/wj9wjyar#jZhCpMmi6A+ogWMv+igCejeroPUtGwAZcQam9IMKEjo
share_updated: 2026-03-17T19:41:04+02:00
---

In previous chapters, we used threads in various ways to perform CPU-bound work. However, not all operations are CPU related. Some require communication with files or other devices, commonly known as I/O operations. These operations do not require CPU usage until the operations complete, at which point thread code continues processing with the result of the I/O operation.

## ==**The IO system**==

The main purpose of the I/O system is to abstract access to physical and logical devices. Accessing a file in the any file system should be different than accessing a serial port, a USB camera or a printer. The I/O System is comprised of multiple components, some in user-mode and most in kernel mode. 

`APP -> IO MANGER -> DRIVER -> HAL`

![[Pasted image 20260316103728.png]]

All file and device operations on the kernel side are initiated by the I/O manager. A request, such as read or write, is handled by creating a kernel structure called I/O Request Packet (IRP), filling in the details of the request and then passing it to the appropriate device driver. 

For real files, this goes to a file system driver, such as NTFS. This process is not essentially different than normal system calls

`APP -> IO MANGER -> IRP -> Driver -> HAL`

![[Pasted image 20260316103808.png]]

As far as the kernel is concerned, I/O operations are always asynchronous. This means that a driver should initiate the operation and return as soon as possible so that the calling thread can regain control. The original caller, however, can choose to make the call synchronous. In that case, the I/O manager waits on behalf of the caller until the operation is done. This flexibility is very convenient from the client’s perspective.

## ==**Files & Devices**==

```
CreateFile() - Create/Open a file object
```
The term “File” used in `CreateFile` is short for “File Object”, which is the abstraction used by the kernel to represent a connection to a device, whether that device happens to be a file in a file system or not

The `lpFileName` parameter indicates which file or device name to create or open. This is not necessarily a “file name”, as indicated by the parameter’s name. It is a symbolic link into the executive’s object manager namespace, with some parsing rules added.

![[Pasted image 20260316104148.png]]

The `dwDesiredAccess` parameter is used to specify the access mask required for accessing the file object. In most cases, you’ll use one or more of the generic access rights

You can also use more fine grained access masks relevant to the file or device being accessed. For example, `FILE_READ_ DATA` is a specific access mask for files, requesting reading their contents.

The `dwCreationDisposition` parameter specifies how to create or open file system objects (files and directories). For other devices, the flag should always be set to `OPEN_EXISTING`. (allows setting three separate flags/values that can be combined by the normal OR operator)

When a program opens a file using `CreateFile`, it can pass **flags** that tell Windows **how the file will be accessed** so the OS can optimize performance.
	`FILE_FLAG_NO_BUFFERING` This **disables the Windows file cache completely**
		Because Windows isn't helping with caching, **your program must respect the disk sector structure**.
		1. Because Windows isn't helping with caching, **your program must respect the disk sector structure**. (likely 512, 4096 bytes ) `GetDiskFreeSpace()`
		2. Memory buffers must start at **sector-aligned addresses**. (`_aligned_malloc`)

```
FlushFileBuffers() - Forces data in cache to be written to file
```

Typically the `WriteFile` and `WriteFileEx` functions write data to an internal buffer that the operating system writes to a disk or communication pipe on a regular basis. The  `FlushFileBuffers` function writes all the buffered information for a specified file to the device or pipe.

Buffering `API -> Internal Buffer -> Disk`
Non-buffering `API -> Disk`
### **Working with Symbolic Links**

A symbolic link is **just a name or path that points to something else** — it can be a file, folder, or device. The OS uses it to let you work with a friendly name while the real target may be somewhere else. Basically just a shortcut name to a longer path

As we’ve seen, `CreateFile` internally works by parsing symbolic links, we can get those links using `QueryDosDevice`

```
QueryDosDevice() - Retrieves information about MS-DOS device names. The function can obtain the current mapping for a particular MS-DOS device name. The function can also obtain a list of all existing MS-DOS device names.
```

if `lpDeviceName` is not NULL, the function looks up the symbolic link and returns its target (if any) in `lpTargetPath`. If `lpDeviceName` is NULL, it returns all symbolic links in `lpTargetPath`, separated by '\0', so they can be iterated if desired

If the function fails because the target buffer is too small, `GetLastError` returns `ERROR_INSUFFICIENT_BUFFER`.

![[Pasted image 20260316111650.png]]
*Symbolic links from winobj*

```
DefineDosDevice() - create a new symbolic link
```
`lpDeviceName` is the symbolic link’s name and `lpTargetPath` is the target of the link.

### **Path Length**

The file name length to CreateFile is traditionally limited to MAX_PATH characters, defined as 260. Windows 10 version 1607 and Windows Server 2016 added a new feature that allows breaking out of these path length limitations. To apply this feature we need 2 settings
- A global registry value named `LongPathsEnabled`  at `HKLM\System\CurrentControlSet\`
- **Control\FileSystem must be set to 1** (DWORD value). The first time an I/O function is called for a process, this value is read and cached for the lifetime of the process. This means any change to this value will be noticed for new processes only. If this value is changed and the system is rebooted, all processes are guaranteed to see the new value.

### **Directories**

The `CreateFile` function can open a handle to an existing directory if the flag `FILE_ FLAG_BACKUP_SEMANTICS` is specified in the `dwFlagsAndAttribute` argument.

```
CreateDirectory/Ex() - Create a new directory (it can have a full path or a relative path)
```
Ex version allows specifying an existing directory from which some properties of the new directory are copied, such as compression and encryption settings

### **File functions**

Once a file handle is open, there is some basic information about the file that can be queried.

```
GetFileSize() - Get File size as a return value
```

`GetFileSize` returns the low 32-bit of the file size as its return value, and the high 32-bit value in the `lpFileSizeHigh` parameter, if specified. If `lpFileSizeHigh` is NULL, the high 32-bit value is not returned. The function returns INVALID_FILE_SIZE in case of an error, defined as 0xffffffff

low 32-bit used if the file size is < 4 GB (file size is less than 4 GB (2³²)) , example:

| File size | nFileSizeHigh | nFileSizeLow |
| --------- | ------------- | ------------ |
| 1 KB      | 0             | 1024         |
| 3 GB      | 0             | 3221225472   |
| 5 GB      | 1             | 1073741824   |
So for files under 4 GB, the **low 32 bits alone represent the full size**.
if file size ≥ 4 GB `HighPart` stores the overflow beyond 32 bits

The maximum file size in practice is much more limited than the theoretical 2 to the 64th power (16 EB) bytes

`ULARGE_INTEGER` struct provided in `WinNT.h` can be used to save a lower and upper bit values
```c
ULARGE_INTEGER size;  
size.LowPart = nFileSizeLow;  
size.HighPart = nFileSizeHigh;  
  
QWORD fileSize = size.QuadPart;
```

```
GetCompressedFileSize() - Get size of a compressed file, it takes a name rather than a handle
```

```
GetFileTime() - Gets basic information about a file relates to its creation, modification and access times
```

```
GetFileAttributes() - Returns file attributes of a file
```

![[Pasted image 20260316210218.png]]

```
GetFileAttributesEx() - just like previous one but returns a WIN32_FILE_ATTRIBUTE_DATA struct containing the info
```
![[Pasted image 20260316210330.png]]

```
GetFileInformationByHandle() - Gets more information of file using handle including size , creating & mod times , attributes (struct BY_HANDLE_FILE_INFORMATION)
```

```
SetFileAttributes() - Used to set attributes to file

The attributes that can be set by this function are: FILE_ATTRIBUTE_ARCHIVE, FILE_ATTRIBUTE_HIDDEN, FILE_ATTRIBUTE_NORMAL, FILE_ATTRIBUTE_NOT_CONTENT_INDEXED, FILE_ATTRIBUTE_OFFLINE, FILE_ATTRIBUTE_READONLY, FILE_ATTRIBUTE_SYSTEM, FILE_ATTRIBUTE_TEMPORARY
```

```
SetFileTime() - set file creation and access times
```

```
SetFileInformationByHandle()
```

## ==**Synchronous I/O**==

The functions that are synchronous, which means the calling thread is now blocked (goes into a wait state) until the operation is complete and the data has been transferred.

When calling `CreateFile` and not specifying `FILE_FLAG_OVERLAPPED` as part of the `dwFlagsAndAttributes` parameter, the file object is created for synchronous I/O only. This is the simplest to work with, so we’ll tackle synchronous I/O first.

The main functions to perform I/O (synchronous or Asynchronous) are `ReadFile` and `WriteFile`, which work with any file object (not necessarily pointing to a file system file)

```
ReadFile() - Read data from file and store in buffer
```

```
WriteFile() - write data from buffer into file
```

The last parameter, `lpOverlapped`, is required to be non-NULL for asynchronous operations, but for Synchronous I/O it should be NULL.

Each file object opened for synchronous access maintains an internal file pointer, that is automatically advanced with each I/O operation. For example, if a file is opened and a read operation of 10 bytes is performed, the file pointer advances by 10 bytes after the operation completes. If another read for 10 bytes is issued, it reads bytes 10 to 19, and the file pointer advances to position 20 in the file.

For sequential reads and writes, that’s great. In some cases, however, you may want to jump forward or back and read/write from a different position. This can be accomplished with one of the following functions:

```
SetFilePointer() - Sets the file pointer to a specific position
```

The functions move the internal file pointer to the desired position. `SetFilePointerEx` is easier to use, since it allows a full 64-bit offset to be provided in the `liDistanceToMove` parameter. `SetFilePointer` accepts the low 32-bit of the offset in `lDistanceToMove` and optionally the high 32-bit in `lpDistanceToMoveHigh`. Both functions attempt to return the previous file pointer

The offset to move, however, is not necessarily the offset from the start of the file. The last parameter, `dwMoveMethod`, indicates how to interpret the provided offset:
	• FILE_BEGIN (0)- from the beginning of the file
	• FILE_CURRENT (1) - from the current file position
	• FILE_END (2) - from the end of the file

```
SetEndOfFile() - sets the end of file to the place where the file pointer is at
```

## ==**Asynchronous I/O**==

The Windows I/O system is asynchronous in nature. Once a device driver issues a request to its controlled hardware (such a disk drive), the driver does not need to wait for the operation to complete. Instead, it marks the request packet as “pending” and returns to its caller. The thread is now free to perform other operations while the I/O is in progress. 

After some time the I/O operation completes by the hardware device. The device issues a hardware interrupt that causes a driver-supplied callback to run and complete the pended request.

It is inconvenient for threads to keep waiting for I/O operations to complete . Asynchronous I/O provides a solution, where a thread initiates a request, and then goes back to serve the next request, and so on, since I/O operations operate concurrently while the CPU executes other code. The only wrinkle in this simplistic model is how a thread is notified of an I/O operation completion.

One of the consequences of a file opened for asynchronous access is that there is no file pointer anymore. This means every operation must somehow provide an offset from the start of the file to perform the operation

We provide an OVERLAPPED structure explained below:
![[Pasted image 20260317040308.png]]

Most important members are the `Offset` , `hEvent` , `Pointer`  

Technically, `Internal` holds the I/O operation’s error code. For an asynchronous operation in progress, it holds `STATUS_PENDING`, which is the kernel equivalent of STILL_ACTIVE (0x103). In fact, Windows defines a macro, `HasOverlappedIoCompleted` that takes `advantage` of this fact.

In an asynchronous operation, the `ReadFile` or `WriteFile` calls normally return FALSE, since the operation is not yet complete, it just started. If the functions return FALSE, then calling `GetLastError` returns `ERROR_IO_PENDING`, it means everything is working as planned

Once the operation completes, how can you tell how many bytes were transferred? There are multiple Ways:
- Thread can wait for the `hEvent` object
- The thread can wait through `GetOverlappedResult`

```
GetOverlappedResult() - accepts the file handle and the OVERLAPPED structure for the particular operation you’re interested in (you can initiate several operations from the same file handle, each having a different OVERLAPPED instance). lpNumberOfBytesTransferred returns the number of bytes actually transferred.
```
The final parameter, `bWait`, specifies whether to wait for the operation to complete (TRUE) before reporting the result or not.

```
GetOverlappedResultEx() - The function allows setting a timeout limit for waiting (dwMilliseconds), and also the ability to wait in an alertable state (bAlertable)
```

![[Pasted image 20260317062855.png]]

```
WriteFileEx - ReadFileEx() - The functions are identical to their non-Ex counterparts, except for an additional argument, which is a function pointer
```

```c
typedef VOID (WINAPI *LPOVERLAPPED_COMPLETION_ROUTINE)(
_In_ DWORD dwErrorCode,
_In_ DWORD dwNumberOfBytesTransfered,
_Inout_ LPOVERLAPPED lpOverlapped
);
```
the callback is wrapped in an Asynchronous Procedure Call (APC) and queued to the original thread that called `ReadFileEx` or `WriteFileEx`. This means this particular thread is the only one which can invoke the callback by getting into an `alertable` state,

```
QueueUserAPC() - Thefunction queues an APCtothetargetthread represented by thehThread parameter, that must have the THREAD_SET_CONTEXT access mask. pfnAPC is a function pointer
```

```c
typedef VOID (WINAPI *PAPCFUNC)(_In_ ULONG_PTR Parameter);
```

This is still an APC, and so the target thread must enter an alertable state if the APC callback is to be executed.

## ==**I/O Completion Ports**==

I/O completion ports deserve their own major section, since they are useful not just for handling asynchronous I/O. We met them briefly when discussing jobs in chapter 4- a job can be associated with an I/O completion port, to receive notifications related to the job.

The I/O completion port’s purpose is to allow processing of completed I/O operations by worker threads, where “worker” here can mean any threads that are bound to the completion port.

An I/O completion port is associated with a file object (could be more than one). It encapsulates a queue of requests, and a list of threads that can serve these requests once completed. Whenever an asynchronous operation completes, one of the threads that waits on the completion ports should wake up and handle the completion, possibly initiating the next request

```
CreateIoCompletionPort() - create an I/O completion port object and associating it with one or more file handles.
```
The function can perform two distinct operations, possibly combining the two. It can do any of the following:
	• Create an I/O completion port not associated with any file objects. 
	• Associate an existing completion port to a file object.
	• Combine the above two operations in a single call.

Function is 1st called with Null , 0 , INVALID_HANDLE_VALUE at the beginning to create an Empty IOCP

Objects created by other functions such as Sockets can also be associated with an I/O completion port.

a completion port is always local to the process that created it. Technically, duplicating such a handle to another process succeeds, but the new handle is unusable.

Once a I/O completion port object is created, it can be associated with one or more file objects (handles).For each file handle, a completion key is specified, which is application defined.

_CompletionKey_ parameter help your application track which I/O operations have completed. This value is not used by **`CreateIoCompletionPort`** for functional control; rather, it is attached to the file handle specified in the `FileHandle` parameter at the time of association with an I/O completion port.

```
GetQueuedCompletionStatus() - The call puts the thread into a wait state until an asynchronous I/O operation initiated with one of the file objects associated with the completion port completes, or the timeout elapsed.
```

Typically `dwMilliseconds` is set to INFINITE, meaning the thread has nothing to do until an I/O operation completes. If the operation that wakes up the thread completed successfully, `GetQueuedCompletionStatus` returns TRUE and the out parameters are filled with the number of bytes transferred, the completion key originally associated with the file handle and the OVERLAPPED structure pointer that was used for the request.

The I/O completion port will not allow more than the maximum threads specified at the port’s creation to succeed the call at the same time.[^1]

Once a thread calls `GetQueuedCompletionStatus` the first time, it becomes bound to the completion port, until the thread exits, the completion port is closed, or the thread calls `GetQueuedCompletionStatus` on a different completion port.

It is possible to manually post a completion packet to an I/O completion port which makes these objects more generic, and not really just about I/O. This is exactly how the notifications work for a job object.

```
PostQueuedCompletionStatus() - post a completion packet to an I/O completion port
```

Remember: **IOCP lets many I/O operations share a small number of threads, instead of assigning a thread per operation.**
![[Pasted image 20260317070838.png]]

Example:
```
1. Issue async recv on many sockets  
2. No threads blocked  
3. OS completes I/O in background  (async)
4. Completion is queued  
5. One of a few threads wakes up and handles it
```

Once `GetQueuedCompletionStatus` finishes waiting, a worker thread dequeues a completed I/O packet from the IOCP queue, and the code following `GetQueuedCompletionStatus` executes as the handler for that completion (i.e., a manual “callback”).

>❌ It’s not “each worker thread does a job for a file”  
   ✅ It’s “the OS completes I/O and hands the result to _any available thread_”

### What an IRP really is

**IRP (I/O Request Packet)** = a **structure in memory** created by the Windows I/O Manager.

It describes:

- what operation (read/write)
- buffer
- size
- file/device
- status

Think of it like:
“do this I/O operation”

1. I/O Manager creates IRP  
2. IRP is passed to drivers (CPU runs driver code)  
3. Driver sends request to hardware  
4. Driver marks IRP pending and returns  
5. Hardware completes operation → triggers interrupt  
6. Driver handles interrupt (ISR/DPC)  
7. Driver completes IRP (`IoCompleteRequest`)  
8. Kernel queues a COMPLETION PACKET to IOCP  
9. Worker thread wakes up (`GetQueuedCompletionStatus`)  
10. Thread processes result (your “callback” code)

**Remark** We can use Struct embedding with the OVERLAP as a 1st member in a struct and when the overlap is filled by the IO MANGER and passed to the worker thread we can get back the struct that hold overlap pointer along with other member as the pointer we pass to the overlap if filled by the IO MANGER and doesn't change the actual pointer to data (The **contents** of `OVERLAPPED` may change, but the **address (pointer)** does not.)

*Notes shape will change , For better explanation*

### **I/O Cancellation**

```c
BOOL CancelIo(
	_In_ HANDLE hFile
)
```
The function attempts to cancel all async operations initiated for the file handle
Parameters:
	- `hFile` - Handle to a File object to close the cancels all pending I/O operations for it

If the function succeeds, the return value is nonzero.

>For more fine-grained control, `CancelIoEx` can be used with a specific OVERLAPPED structure representing the operation to cancel.

In any case, canceling an I/O operation is not guaranteed to succeed. The canceling operation itself is implemented by the device driver responsible for the operation. Some drivers (especially for devices) don’t support cancellation at all. Even if the driver supports cancelation, it may not be able to do so for every operation.

>If an I/O operation is canceled successfully, the return value from `GetLastError` (or the error result provided by a thread pool callback) is `ERROR_OPERATION_ABORTED`.

```c
BOOL CancelSynchronousIo(
	_In_ HANDLE hThread
);
```
The function cancels sync I/O operations initiated by another thread 
Parameters:
	- `hThread` - Handle to thread

the operation returns FALSE and `GetLastError` returns `ERROR_OPERATION_ABORTED`.

>the thread that initiated the request cannot cancel it, since it’s waiting for the I/O to complete so another thread cancels the operation.


## ==**Devices**==

Working with devices (that is, non file-system files), is essentially no different than working with file-system files. The `ReadFile` and `WriteFile` functions work for any device, including asynchronously, although not all devices support read and write operations.

```c
BOOL DeviceIoControl(
_In_ HANDLE hDevice,
_In_ DWORD dwIoControlCode,
_In_ LPVOID lpInBuffer,
_In_ DWORD nInBufferSize,
_Out_ LPVOID lpOutBuffer,
_In_ DWORD nOutBufferSize,
_Out_opt_ LPDWORD lpBytesReturned,
_Inout_opt_ LPOVERLAPPED lpOverlapped
);
```
`DeviceIoControl` is a general-purpose function that allows sending a request for a device through **The IOCTL Gateway** For operations that are device-specific (like ejecting a disk or querying battery status) 
Parameters:
	- `hDevice` - A handle to the device on which the operation is to be performed.
	- `dwIoControlCode` - The control code for the operation. This value identifies the specific operation to be performed and the type of device on which to perform it.
	- `lpInBuffer` - buffer sent to the device
	- `lpOutBuffer` - buffer received from device

>`CreateFile` works with any symbolic link For example, there are symbolic links called “PhysicalDrive0” and perhaps others, which is a way to open a drive’s sectors directly, without looking at it through the file system’s lens.

symbolic names can be used to directly communicate with a driver instead of using `DeviceIoControl`

#### **Plug and Play (PnP) & The DevNode**

**Plug and Play (PnP)** is the combination of hardware and software support that enables Windows to recognize and adapt to hardware changes with little to no user intervention.

- **DevNode (Device Node):** A dynamic internal structure maintained by the **PnP Manager**. It represents a device instance in the system.
    
- **Device Instance ID:** A unique, system-wide string assigned to each DevNode (e.g., `USB\VID_045E&PID_00DB\5&2C0D01F&0&1`).
	
- **Enumeration:** The process where a "Bus Driver" (like USB or PCI) detects a new device and tells the PnP Manager to create a new DevNode.

### **GUIDs , Interfaces , Classes**

Another interesting set of symbolic links look like nonsense strings that could not have been selected by humans; indeed, these were generated by the kernel to (at least) ensure uniqueness. These weird-looking symbolic links are used for hardware device names. For example, if you want to access a camera connected to your computer, how would you do it?

Behind the scenes, devices expose **device interfaces**, which you can think of as being similar to software interfaces. Each interface represents some functionality. For example, a printer device can “implement” a printing interface and a scanning interface. With these interfaces, you can search “printers” or “scanners”.

Device interfaces are represented by GUIDs, and many are defined by Microsoft and can be found in the documentation. This means we need to use an API to locate a device, and part of the information returned by the API, is the device’s symbolic link that we can pass as-is to `CreateFile`

Beside device interfaces , windows add more layers of filtering to identify the device better such as `catigory` where similar devices are **grouped** together this is called **Device Setup Class**. Also devices have a class that represents “How can software _access_ this device?” called a **Device Interface Class** A device interface class is a way of exporting device and driver functionality to other system components

**Device Setup Class** – A group of all devices that share the same category or type, identified by a class GUID. Example: `all keyboards`, `all mice`, `all USB controllers`. It’s mainly used for **driver installation, grouping, and enumeration**

**Device Interface Class** – Defines a **way to communicate with a specific device**. Each device can expose one or more interfaces, each with its own GUID, which user-mode applications or other drivers can use to access the device’s capabilities (e.g., keyboard input, lighting control). Also provide a mechanism for grouping device interfaces according to shared characteristics or functionality.

>Setup classes govern how the operating system installs and configures devices. Interface classes enable runtime communication and functionality between drivers, applications, and devices.

A device often belongs to a setup class and exposes several device interfaces in different interface classes at the same time. Nevertheless, the two types of classes serve different purposes and are not interchangeable.

![[Pasted image 20260321172908.png]]

Take:
- `GUID_DEVINTERFACE_DISK`
This is the **interface class**.
Inside it, you might have multiple **device interfaces** like:
- `\\?\PhysicalDrive0`
- `\\?\PhysicalDrive1`
Each one represents a **specific device instance**, but they all belong to the same class.
#### Device interface classes 

A device interface class is a way of exporting device and driver functionality to other system components, including other drivers, as well as user-mode applications. A driver can register and enable a _device interface_ instance of the _device interface class_ for each device object to which user-mode I/O requests might be sent. Each _device interface class_ should represent a conceptual functionality that any _device interface_ in that class should support or represent such as a particular I/O contract.

Each device interface class is associated with a GUID. The system defines GUIDs for common device interface classes in device-specific header files. Vendors can create additional device interface classes.

>For example, three different types of mouse devices could register _device interfaces_ that are members of the same device interface class, even if one connects through a USB port, a second through a serial port, and the third through an infrared port. Each driver registers its device as a member of the interface class GUID_DEVINTERFACE_MOUSE

#### Device Information Sets
In user mode, devices that belong to either device setup classes or device interface classes are managed by using _device information elements_ and _device information sets._

A device information set consists of device information elements for all the devices that belong to some device setup class or device interface class.

Each device information element contains a handle to the device's _devnode_, and a pointer to a linked list of all the device interfaces associated with the device described by that element.

Device Information Element This represents **a single device** within a device information set. You usually iterate through the `HDEVINFO` set to get each device element

![[Pasted image 20260321155639.png]]

```c
WINSETUPAPI HDEVINFO SetupDiCreateDeviceInfoList(
  [in, optional] const GUID *ClassGuid,
  [in, optional] HWND       hwndParent
);
```
 This function allows us to create a device information set
 Parameters:
	- `ClassGuid` - A pointer to the **GUID** of the device setup class to associate with the newly created device information set.
	- `HWND` - A handle to the top-level window to use for any user interface that is related to non-device-specific actions (such as a select-device dialog box that uses the global class driver list). This handle is optional and can be **NULL**.

>Once we created an empty device information set, we can add some device information elements

> A **device instance ID** is a system-supplied device identification string that uniquely identifies a device in the system. The Plug and Play (PnP) manager assigns a device instance ID to each device node (_devnode_) (This info needed bellow)

```c
WINSETUPAPI HDEVINFO SetupDiGetClassDevs(
	[in, optional] const GUID *ClassGuid,
	[in, optional] PCWSTR Enumerator,
	[in, optional] HWND hwndParent,
	[in] DWORD Flags
);
```
The function returns a handle to a device information set that contains requested device information elements for a local computer.
Parameters:
	- `ClassGuid` - A pointer to an optional GUID for a device of a interface class If you pass a valid GUID → it **filters devices** to that setup class or interface class. if you pass `NULL` → it includes **all classes** (depending on flags).
	- `Enumerator` - This pointer is optional and can be **NULL**. If an _enumeration_ value is not used to select devices, set _Enumerator_ to **NULL**
	- `hwndParent` - This handle is optional and can be **NULL**.
	- `Flags` - A variable of type DWORD that specifies control options that filter the device information elements that are added to the device information set.

If the operation succeeds, the function returns a handle to a device information set that contains all installed devices that matched the supplied parameters.

>To enumerate all device interfaces across all classes set on the local machine we can use it as `SetupDiGetClassDevs(NULL, NULL, NULL, DIGCF_ALLCLASSES)` 

>To discover all interface classes on the system, you call `SetupDiGetClassDevs` with `NULL` and the flags `DIGCF_ALLCLASSES | DIGCF_DEVICEINTERFACE`, it returns all setup classes then enumerate all device interfaces using `SetupDiEnumDeviceInterfaces`. Each returned interface contains its `InterfaceClassGuid`, which allows you to infer and group interface classes dynamically.


```c
WINSETUPAPI BOOL SetupDiCreateDeviceInfoA(
  [in]            HDEVINFO         DeviceInfoSet,
  [in]            PCSTR            DeviceName,
  [in]            const GUID       *ClassGuid,
  [in, optional]  PCSTR            DeviceDescription,
  [in, optional]  HWND             hwndParent,
  [in]            DWORD            CreationFlags,
  [out, optional] PSP_DEVINFO_DATA DeviceInfoData
);
```
This function creates a new device information element and adds it as a new member to the specified device information set.
Parameters:
	- `HDEVINFO` - Handle to the empty device information set we made
	- `DeviceName` - A pointer to a NULL-terminated string that supplies either a full device instance id or root-enumerated device ID without the enumerator prefix and instance identifier suffix
	- `DeviceDescription` - A pointer to a NULL-terminated string that supplies the text description of the device.
	- `hwndParent` - A handle to the top-level window to use for any user interface that is related to installing the device.
	- `CreationFlags` - A variable of type DWORD that controls how the device information element is created
	- `DeviceInfoData` - A pointer to a `SP_DEVINFO_DATA` structure that receives the new device information element. This pointer is optional and can be **NULL**. If the structure is supplied, the caller must set the **`cbSize`** member of this structure to **`sizeof(SP_DEVINFO_DATA)`** before calling the function.

We can also enumerate device **information** in a device information set 

```c
WINSETUPAPI BOOL SetupDiEnumDeviceInfo(
  [in]  HDEVINFO         DeviceInfoSet,
  [in]  DWORD            MemberIndex,
  [out] PSP_DEVINFO_DATA DeviceInfoData
);
``` 
This function enumerates all devices that belong to the information set that meet certain criteria.
Parameters:
	- `HDEVINFO` - Handle to the empty device information set we made
	- `MemberIndex` - A zero-based index of the device information element to retrieve.
	- `DeviceInfoData` - A pointer to an `SP_DEVINFO_DATA` structure to receive information about an enumerated device information element. The caller must set `DeviceInfoData.cbSize` to `sizeof(SP_DEVINFO_DATA)`.

>SP_DEVINFO_DATA contains the GUID of the class that the device belongs to and a _device instance_ handle that points to the devnode for the device.


We can now enumerate the device's **interfaces** (We use device interface GUID ex `GUID_DEVINTERFACE_DISK` not setup class GUID) in a device information set 

```c
WINSETUPAPI BOOL SetupDiEnumDeviceInterfaces(
  [in]           HDEVINFO                  DeviceInfoSet,
  [in, optional] PSP_DEVINFO_DATA          DeviceInfoData,
  [in]           const GUID                *InterfaceClassGuid,
  [in]           DWORD                     MemberIndex,
  [out]          PSP_DEVICE_INTERFACE_DATA DeviceInterfaceData
);
```
the function enumerates the device interfaces that are contained in a device information set.
Parameters:
	- `HDEVINFO` - Handle to the empty device information set we made
	- `PSP_DEVINFO_DATA` - A pointer to an `SP_DEVINFO_DATA` structure that specifies a device information element, If this parameter is **NULL**  the function return information about the interfaces that are associated with all the device information elements
	- `InterfaceClassGuid` - A pointer to a GUID that specifies the device interface class for the requested interface.
	- `MemberIndex` - A zero-based index into the list of interfaces in the device information set
	- `DeviceInterfaceData` - A pointer to a caller-allocated buffer that contains, on successful return, a completed `SP_DEVICE_INTERFACE_DATA` structure that identifies an interface that meets the search parameters. The caller must set `DeviceInterfaceData.cbSize` to `sizeof(SP_DEVICE_INTERFACE_DATA)` before calling this function.

After getting the interface class associated with a device , we can then procced to get all the interface info:
```c
WINSETUPAPI BOOL SetupDiGetDeviceInterfaceDetailA(
  [in]            HDEVINFO                           DeviceInfoSet,
  [in]            PSP_DEVICE_INTERFACE_DATA          DeviceInterfaceData,
  [out, optional] PSP_DEVICE_INTERFACE_DETAIL_DATA_A DeviceInterfaceDetailData,
  [in]            DWORD                              DeviceInterfaceDetailDataSize,
  [out, optional] PDWORD                             RequiredSize,
  [out, optional] PSP_DEVINFO_DATA                   DeviceInfoData
);
```
The function returns details about a device interface.
Parameters:
	- `HDEVINFO` device info set handle
	- `PSP_DEVICE_INTERFACE_DATA` - pointer to device interface data struct 
	- `PSP_DEVICE_INTERFACE_DETAIL_DATA_A` - pointer to interface detail struct
	- `DWORD` - size of `PSP_DEVICE_INTERFACE_DETAIL_DATA_A`
	- `PDWORD` - pointer to required size
	- `PSP_DEVINFO_DATA` - pointer to device info struct

>Notice that we always get the class of the object then get its detail (in devices and interfaces we get the class then get the details of the object) , when using **SetupDiGetDeviceInterfaceDetailA** we 1st set  pointer to detail A to null and size to 0 then get the size, allocate on the heap and cast it to `PSP_DEVICE_INTERFACE_DETAIL_DATA_A` then use that allocated struct in the function

To show a human-readable name (like "Logitech USB Mouse") of a device instance:
```c
WINSETUPAPI BOOL SetupDiGetDeviceRegistryProperty (
  [in]            HDEVINFO         DeviceInfoSet,
  [in]            PSP_DEVINFO_DATA DeviceInfoData,
  [in]            DWORD            Property,
  [out, optional] PDWORD           PropertyRegDataType,
  [out, optional] PBYTE            PropertyBuffer,
  [in]            DWORD            PropertyBufferSize,
  [out, optional] PDWORD           RequiredSize
);
```
the function retrieves a specified Plug and Play device property.
Parameters:
	- `HDEVINFO` - Handle to the device information set
	- `PSP_DEVINFO_DATA` - Struct holding device interface info
	- `Property` - specifies the property to be retrieved
	- `PropertyRegDataType` - A pointer to a variable that receives the data type of the property that is being retrieved.
	- `PropertyBuffer` - A pointer to a buffer that receives the property that is being retrieved. If this parameter is set to **NULL**, and _PropertyBufferSize_ is also set to zero, the function returns the required size for the buffer in _RequiredSize_.
	- `PropertyBufferSize` - The size, in bytes, of the _PropertyBuffer_ buffer.
	- `RequiredSize` - A pointer to a variable of type DWORD that receives the required size


After having a device info struct which has a GUID of a device, we can get the device setup class name:
```c
WINSETUPAPI BOOL SetupDiClassNameFromGuidA(
  [in]            const GUID *ClassGuid,
  [out]           PSTR       ClassName,
  [in]            DWORD      ClassNameSize,
  [out, optional] PDWORD     RequiredSize
);
```
The function retrieves the class name associated with a class GUID.
Parameters:

	- `ClassGuid` - GUID of the device we want to get it's setup class
	- `ClassName` - pointer to hold class name
	- `ClassNameSize` - size of the buffer
	- `RequiredSize` - required size for the buffer


![[Pasted image 20260322012558.png]]

```
Setup Class (category)
↓
Device Information Set (many devices)
↓
Device (one real object)
↓
Interface (how to use it)
↓
CreateFile → HANDLE
```

>To enum all setup classes in the local machine , you 1st get all devices using `
>`SetupDiGetClassDevs(NULL, NULL, NULL, DIGCF_ALLCLASSES)` this returns a handle to an information set holding all devices , then we use those devices to get the setup class of them , we loop infinitely with index and pass it to `SetupDiEnumDeviceInfo` which returns device information including its GUID and it fails when the loop ends so we break , then we use the GUID with `SetupDiClassNameFromGuidA` to get the setup class name 

## ==**Pipes and Mailslots**==

Pipes is uni- or bi-directional communication mechanism, that works across processes and across machines on the network. Mailslots is a uni-directional communication mechanism, that works locally or over the network.

#### **pipes**

Pipes come in two variants- anonymous and named. Anonymous pipes is a simple uni-direction communication mechanism that is limited to the local machine.

We can create an anonymous pipe using:
```c
BOOL CreatePipe(
  [out]          PHANDLE               hReadPipe,
  [out]          PHANDLE               hWritePipe,
  [in, optional] LPSECURITY_ATTRIBUTES lpPipeAttributes,
  [in]           DWORD                 nSize
);
```
Creates an anonymous pipe, and returns handles to the read and write ends of the pipe.
Parameters:
	- `hReadPipe` - A pointer to a variable that receives the read handle for the pipe.
	- `hWritePipe` - A pointer to a variable that receives the write handle for the pipe.
	- `lpPipeAttributes` - A pointer to a `SECURITY_ATTRIBUTES` structure
	- `nSize` - The size of the buffer for the pipe, If this parameter is zero, the system uses the default buffer size.

>A classic example of using anonymous pipes is to redirect input and/or output to **child** process. This allows one process to feed data to another process, while the other process has no idea, and doesn’t really care, it just uses the standard handle(s) for input/output. 

once we create an anonymous pipe, we need to pass the HANDLE responsible for receive or transfer to process we want to communicate with , so handles created should be inheritable for the child process to be able to use them. Mark handles as inheritable (`SECURITY_ATTRIBUTES` with `bInheritHandle = TRUE`).

The STARTUPINFO structure (used in `CreateProcess`) is initialized by setting the `hStdOutput` member to the value of the write handle , meaning any operation on the stdout file will be redirected to the pipe line to the owner process , This works because the new process gets a valid handle referring to the same kernel object. The flag `STARTF_USESTDHANDLES` ensures the standard handles are picked up automatically by the new process.

>`STARTUPINFO` is a struct where you can specify:
>- what the child process should use as:
    - **stdin**
    - **stdout**
    - **stderr**

If we want to pass the handle to a process that isn't a child and is already running , we can use `DuplicateHandle` to duplicate the handle to the target process and it becomes able to use it

Named pipes and maillots are discussed in their own chapter (in Part 2)

## ==**Transactional NTFS**==

Transactional NTFS is a system that windows introduced allowing multiple File operations to either all succeed or all fail for example:
- create File A ✅
- Write File B ✅
- delete File C ❌ (failed)
then all the operations fail since C failed , this adds a layer of consistency and atomicity allowing us to take action only if every thing went as planned.

**Transactional NTFS mainly applies to write operations** (create, modify, delete).
**Reads don’t typically participate in transactions** in the same meaningful way.
- A failed _read_ doesn’t usually trigger rollback unless it’s part of a broader transactional context enforced by the API.

>The Windows executive has a component called Kernel Transaction Manager (KTM), that provides support for using transactions for file (and registry) operations.

A transaction is a set of operations that adhere to the so-called ACID properties:
	• Atomicity - either all operations in a transaction succeed, or all operations fail.
	• Consistency - the file system will always be in a consistent state.
	• Isolation - multiple transaction in progress do not affect each other.
	• Durability - system failure should not cause the transaction to violate the earlier properties.

Inorder to use Transactional NTFS , we 1st create a new transaction object.
```c
HANDLE CreateTransaction(
  [in, optional] LPSECURITY_ATTRIBUTES lpTransactionAttributes,
  [in, optional] LPGUID                UOW,
  [in, optional] DWORD                 CreateOptions,
  [in, optional] DWORD                 IsolationLevel,
  [in, optional] DWORD                 IsolationFlags,
  [in, optional] DWORD                 Timeout,
  [in, optional] LPWSTR                Description
);
```
Creates a new transaction object.
Parameters:
	- `LPSECURITY_ATTRIBUTES` - A pointer to a [SECURITY_ATTRIBUTES](https://learn.microsoft.com/en-us/windows/win32/api/wtypesbase/ns-wtypesbase-security_attributes) structure that determines whether the returned handle can be inherited by child processes.
	- `LPGUID` - Reserved , 0
	- `CreateOptions` - can be zero or `TRANSACTION_DO_NOT_ PROMOTE` to prevent promoting the transaction to a distributed one.
	- `IsolationLevel` - Reserved; specify zero (0).
	- `IsolationFlags` - Reserved; specify zero (0).
	- `Timeout` - the transaction will abort after the specified time in milliseconds elapses.
	- `Description` - optional human-readable string describing the transaction
  
With a transaction handle in hand, several file related functions accept a transaction handle

```c
HANDLE CreateFileTransactedA(
  [in]           LPCSTR                lpFileName,
  [in]           DWORD                 dwDesiredAccess,
  [in]           DWORD                 dwShareMode,
  [in, optional] LPSECURITY_ATTRIBUTES lpSecurityAttributes,
  [in]           DWORD                 dwCreationDisposition,
  [in]           DWORD                 dwFlagsAndAttributes,
  [in, optional] HANDLE                hTemplateFile,
  [in]           HANDLE                hTransaction,
  [in, optional] PUSHORT               pusMiniVersion,
                 PVOID                 lpExtendedParameter
);
```
Creates or opens a file, file stream, or directory as a transacted operation. The function returns a handle that can be used to access the object.
Parameters:
	- `lpFileName` - The name of an object to be created or opened.
	- `dwDesiredAccess` - The access to the object
	- `dwShareMode` - The sharing mode to the object
	- `lpSecurityAttributes` - pointer to security attributes struct
	- `dwCreationDisposition` - An action to take on files that exist and do not exist.
	- `dwFlagsAndAttributes` - The file attributes and flags
	- `hTemplateFile` - NULL
	- `hTransaction` - A handle to the transaction object
	- `pusMiniVersion` - The pusMiniVersion parameter should be NULL if the file is opened for read access only. If opened for write access, it indicates what kind of view the file should present to clients during the transaction
	- `lpExtendedParameter` - NULL
  
The handle returned by `CreateFileTransacted` can be passed to normal I/O access functions, 
>such as `ReadFile` and `WriteFile`. This means that once a file object is created transacted, all the other operations on the file remain exactly the same.

Once we added all the operations we want to execute , we commit the Transaction object and execute all
```c
BOOL CommitTransaction(
  [in] HANDLE TransactionHandle
);
```
Requests that the specified transaction be committed.
Parameters:
	- `TransactionHandle` - A handle to the transaction to be committed.

If something went wrong with the various operations in the transaction, you can request all operations to roll back:
```c
BOOL RollbackTransaction(
  [in] HANDLE TransactionHandle
);
```
Requests that the specified transaction be rolled back.
Parameters:
	- `TransactionHandle` - A handle to the transaction to be committed.

>Transaction handles are closed normally with `CloseHandle`. The function is synchronous.

![[Pasted image 20260331172423.png]]

## ==**File Search and Enumeration**==

Sometimes there is a need to search or enumerate for files and directories. Fortunately, the file management APIs provide several functions to accomplish such a fit.

We can find a file or 1st file in directory using:
```c
HANDLE FindFirstFileA(
  [in]  LPCSTR             lpFileName,
  [out] LPWIN32_FIND_DATAA lpFindFileData
);
```
Parameters:
	- `lpFileName` - The directory or path, and the file name. The file name can include wildcard characters, for example, an asterisk (*) or a question mark (?).
	-`LPWIN32_FIND_DATAA` - A pointer to the `WIN32_FIND_DATA` structure that receives information about a found file or directory.

The functions return a search handle, which is INVALID_HANDLE_VALUE if an error occurs.
Each result is returned with the `WIN32_FIND_DATA` structure defined like so:
![[Pasted image 20260331194540.png]]

To find the next file using the search handle we use:
```c
BOOL FindNextFileA(
  [in]  HANDLE             hFindFile,
  [out] LPWIN32_FIND_DATAA lpFindFileData
);
```
Continues a file search from a previous call to the `FindFirstFile`
Parameters:
	- `hFindFile` - Search handle
	-`LPWIN32_FIND_DATAA` - A pointer to the `WIN32_FIND_DATA` structure that receives information about a found file or directory.

# Functions discussed in this chapter:
```
CreateFile
CreateDirectory

GetFileSize
GetFileTime
GetFileAttributes
GetCompressedFileSize
GetFileInformationByHandle

SetFileAttributes
SetFileTime
SetFileInformationByHandle

ReadFile
WriteFile
SetEndOfFile
SetFilePointer

GetOverlappedResult
QueueUserAPC

CreateIoCompletionPort
GetQueuedCompletionStatus
PostQueuedCompletionStatus

DefineDosDevice
QueryDosDevice

FlushFileBuffers

CancelIo
CancelSynchronousIo

SetupDiCreateDeviceInfoList
SetupDiGetClassDevs
SetupDiCreateDeviceInfoA
SetupDiEnumDeviceInfo
SetupDiEnumDeviceInterfaces
SetupDiGetDeviceInterfaceDetailA
SetupDiGetDeviceRegistryProperty
SetupDiClassNameFromGuidA

CreatePipe

CreateTransaction
CreateFileTransactedA
CommitTransaction
RollbackTransaction

FindFirstFileA
FindNextFileA
```

---
# Footer

[^1]: However, if a thread for which `GetQueuedCompletionStatus` succeeded, and that thread, while processing the completion operation enters a wait state for whatever reason (`SuspendThread`, `WaitForSingleObject`. etc.), the completion port will allow another thread to have its `GetQueuedCompletionStatus` call end its wait.

[^2]: “SetupDi” is short for “Setup Device Interface”.
