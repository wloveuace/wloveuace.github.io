---
share_link: https://share.note.sx/mrs3ie3q#9XHtGXlpo4bGNWk4XFsDMxxq/hMMS8M7MZ4oRCmNT5E
share_updated: 2026-02-08T17:23:02+02:00
---

# ==Kernel Objects , Handles & Sharing ==

Windows is an object-based operating system, exposing various types of objects (usually referred to as kernel Objects), that provide the bulk of the functionality in Windows. Example object types are processes, threads and files.[^1] , Kernel object types that are exported to user mode via the Windows API. `Examples: mutex, semaphore, file, process, thread, timer`

Since kernel objects reside in system space, they cannot be accessed directly from user mode. Applications must use an indirect mechanism to access kernel objects, known as handles.

==**Handles**== are logically an index to an array of entries in a handle table maintained on a process by process basis, that points to a kernel object residing in system space. Handles provide access to kernel objects to user mode, also provides multiple ways to keep User mode code safe:

- Any change in the object type’s data structure in a future Windows release will not affect any client.
- Access to the object can be controlled through security access checks
- Handles are private to a process ,so having a handle to a particular object in one process means nothing in another process context.
[^2]

`Kernel object attributes - diagram 1`
![[WSP1/Chapter 2/d2.png]]

```
CloseHandle() - used to close a handle ( Once an object used by a user mode client is no longer needed, the client code should close the handle used to access the object )
```
[^3]

There are various Create* and Open* functions to create/open objects and retrieve back handles to these objects. If the object cannot be created or opened, the returned handle is in most cases NULL (0). [^4]

If the function succeeds and a name was provided ,the returned handle can be to a new handle or to an existing handle with that name. The code can check this by calling `GetLastError` and comparing the result to `ERROR_ALREADY_EXISTS`. If it is, then it’s not a new object, but rather another handle to an existing object. This is one of those rare cases where `GetLastError` can be called even if the API in question succeeded

==**Handle Structure**== a handle points indirectly to a small data structure in kernel space that holds a few pieces of information for that handle.

On 32bit systems ,this handle entry is 8bytes in-size ,and it’s 16 bytes in size on 64 bit systems (technically 12 bytes are enough, but it’s extended to 16 bytes for alignment purposes). Each entry has the following ingredients

- Pointer to the actual object. Since the lower bits are used for flags and to improve CPU access times by address alignment, an object’s address is multiple of 8 on 32 bit systems and multiple of 16 on 64 bit systems.
- Access mask, indicating what can be done with this handle. In other words, the access mask is the power of the handle.
- Three flags: Inheritance, Protect from close and Audit on close 

The access mask is a bitmask, where each “1” bit indicating a certain operation that can be carried using that handle. The access mask is set when the handle is created by creating an object or opening an existing object. If the object is created, then the caller typically has full access to the object. But if the object is opened, the caller needs to specify the required access mask, which it may or may not get.

`Handle structure - diagram 2`
![[WSP1/Chapter 2/d1.png]]

```
SetHandleInformation() - used when changing the inheritance and protectionflags of a handle ( setting flag to 0 will reset it )

GetHandleInformation() - used to get handle flags information
```
[^5]

==**Object Security**== when creating an object, security features can be applied in the Create function ( in the security attribute struct ), The main member that is really about security is `lpSecurityDescriptor`, which can point to a security descriptor object, which essentially specifies who-can-do-what with the object[^6]

==**Object Names**== Some types of objects can have string-based names. These names can be used to open objects by name with a suitable Open function. (Note that not all objects have names). Calling a Create function with a name creates the object with that name if an object with that name does not exist, but if it exists it just opens the existing object.

==**Sharing Kernel Objects**== some mechanism must be in place to allow such sharing. In fact, there are three:
- Sharing by name
- Sharing by handle inheritance
- Sharing by duplicating handles

```
DuplicateHandle() - used to duplicate a handle named/unnamed, (one concern is that handle to source object is required)
```

==**Private Object Namespaces**== this mechanism allows sharing the object to only specified process, where the object’s name is now under a private object namespace and is not visible to all.Creating a private namespace is a two-step process. First, a helper object called a Boundary Descriptor must be created. This descriptor allows adding certain Security IDs (SIDs) that would be able to use private namespaces created based on that boundary descriptor. This can help tighten security on the private namespace(s).

```
CreateBoundaryDescriptor() - used to create a boundary descriptor
```

Once a boundary descriptor exists, two functions can be used to restrict access to any private namespace created through that descriptor ( AddSIDToBoundaryDescriptor and AddIntegrityLabelToBoundaryDescriptor )

```
AddSIDToBoundaryDescriptor() - the SID is typically a group’s SID, allowing all users in that group access to the private namespaces

AddIntegrityLabelToBoundaryDescriptor() - allows setting a minimum integrity level for processes that wish to open objects in private namespace managed by this boundary descriptor.

CreatePrivateNamespace() - create the actual private namespace
OpenPrivateNamespaceW() - get private namespace handle
ClosePrivateNamespace() - close private name space
```

Once the namespace is created or opened, named objects can be created normally with the nameinthe formalias\name where“alias” is the `lpAliasPrefix` parameter from creating or opening the namespace.


#### Functions discussed in this chapter
```
CloseHandle()
Open/Create Handle object() *this is just a pattern*

SetHandleInformation()
GetHandleInformation()

DuplicateHandle()

CreateBoundaryDescriptor()
AddSIDToBoundaryDescriptor()
AddIntegrityLabelToBoundaryDescriptor()

CreatePrivateNamespace()
OpenPrivateNamespaceW()
ClosePrivateNamespace()
```

---

[^1]: Object Manager (part of the Executive) when requested to do so by user or kernel mode code. kernel objects are reference counted, so only when the last reference to the object is released will the object be destroyed and freed from memory

[^2]: Kernel objects are reference counted. The Object Manager maintains a handle count and a pointer count, the sum of which is the total reference count for an object (direct pointers can be obtained from kernel mode).

[^3]: Trying to access the object through the closed handle will fail, with GetLastError returning ERROR_INVALID_HANDLE (6). The client does not know, in the general case, whether the object has been destroyed or not. The Object Manager will delete the object if its reference drops to zero

[^4]: One notable exception to this rule is the CreateFile function that returns INVALID_HANDLE_VALUE (-1) if it fails.

[^5]: pseudo-handles, although they are used just like any other handle when needed. Calling CloseHandle on pseudo-handles always fails.

[^6]: Passing NULL for SECURITY_ATTRIBUTES leaves the inheritance bit clear.
