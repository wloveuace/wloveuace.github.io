---
share_link: https://share.note.sx/im88fd30#QlehzUTGkPN8mOcwpXDpszYPObqN0JgAaB2Wmj+ga1g
share_updated: 2026-02-08T17:12:57+02:00
---

# ==**Jobs**==

Jobs allow user to manage one or more processes. Most of their capability revolves around limiting the managed processes in some ways.

```
CreateJobObject() - creates a job object 
OpenJobObject() - open object
```

`job handle access mask - diagram 1`
![[WSP1/Chapter 4/d1.png]]

```
AssignProcessToJobObject() - assign a process to a job object
```

Once a process is associated with a job, it cannot break out. If that process creates a child process, the child process is created by default as part of the parent’s process job.[^1][^2]

==**Nested Jobs**== This makes jobs much more useful than they used to, since if a process you wish to control with a job was already part of a job- there was no way to associate it with another job. A process that is assigned a second job, causes a job hierarchy to be created (if possible). The second job becomes a child of the first job. The basic rules are the following:

1. A limit imposed by a parent job affects the job and all child jobs (and hence all processes in those jobs).
2. Any limit imposed by a parent job cannot be removed by a child job ,but it can be more strict

`Nested jobs - diagram 2`
![[WSP1/Chapter 4/d2.png]]
[^3]

```
QueryInformationJobObject() - query information about a job object
```

`JobObjectInfoClass flags enum  - diagram 3`
![[WSP1/Chapter 4/d3.png]]
[^4]
```
TerminateJobObject() - terminate job object
```

```
SetInformationJobObject() - set job object information ( just like the query function , you specify a enum type and fill the struct that you want to apply and give it to the function )
```

==**Job Notifications**== When job limits are violated, or when certain events occur, the job can notify an interested party via an I/O completion port, that can be associated with the job. I/O completion ports are typically used to handle completion of asynchronous I/O operations

```
CreateIoCompletionPort() - create I/O compelation port [^5]
GetQueuedCompletionStatus() - wait for the completion port to fire[^6]
```

#### Functions discussed in this chapter 
```
CreateJobObject()
OpenJobObject()
AssignProcessToJobObject()
TerminateJobObject()

QueryInformationJobObject()
SetInformationJobObject()

CreateIoCompletionPort()
GetQueuedCompletionStatus()
```

---

[^1]: The CreateProcess call includes the CREATE_BREAKAWAY_FROM_JOB flag and the job allows breaking out of it

[^2]: The job has the limit flag JOB_OBJECT_LIMIT_SILENT_BREAKAWAY_OK. In this case, any child process is created outside the job without requiring any special flags.

[^3]: changes in Job 1 will be applied on Job 2 aswell

[^4]: each enum value to query has a STRUCT used in the LPVOID 3rd parameter as a return value based on the chosen enum, the table shows the enum values and the appropriate struct to use with it

[^5]: Normally the first argument to CreateIoCompletionPort is a file handle, but in this case it’s INVALID_HANDLE_VALUE, indicating no file is associated with the I/O completion port.

[^6]: JOBOBJECT_ASSOCIATE_COMPLETION_PORT must be set with port and key ( in SetInformationJobObject )
	
	```
	HANDLE hPort = CreateIoCompletionPort(
	    INVALID_HANDLE_VALUE,
	    NULL,
	    0,
	    1
	);
	
	JOBOBJECT_ASSOCIATE_COMPLETION_PORT assoc = {0};
	assoc.CompletionKey  = (PVOID)1;   // any value you want
	assoc.CompletionPort = hPort;
	
	SetInformationJobObject(
	    hJob,
	    JobObjectAssociateCompletionPortInformation,
	    &assoc,
	    sizeof(assoc)
	);
	
	
	DWORD msg;
	ULONG_PTR key;
	LPOVERLAPPED ov;
	
	BOOL ok = GetQueuedCompletionStatus(
	    hPort,
	    &msg,
	    &key,
	    &ov,
	    INFINITE
	);
	
	msg objects:
	JOB_OBJECT_MSG_NEW_PROCESS
	JOB_OBJECT_MSG_EXIT_PROCESS
	JOB_OBJECT_MSG_ACTIVE_PROCESS_ZERO
	JOB_OBJECT_MSG_ACTIVE_PROCESS_LIMIT
	JOB_OBJECT_MSG_PROCESS_MEMORY_LIMIT
	JOB_OBJECT_MSG_JOB_MEMORY_LIMIT
	JOB_OBJECT_MSG_END_OF_JOB_TIME
	
	```
	
