---
share_link: https://share.note.sx/gzqbrkq1#m8fw0pDxafvXXCqnRRz7SXRtoY8K9QfN24r8llzmoYQ
share_updated: 2026-02-08T17:22:33+02:00
---

# ==**The Process**==

Processes in windows have multiple types, although the everyday process didn't have its structure changed since windows NT, but new process environments have been released:

- Protected Processes - These processes were introduced in Windows Vista. They were created to support Digital Rights Management (DRM) protection by preventing intrusive access to processes rendering DRM-protected content.

- UWP Processes - These processes, available starting with Windows 8, host the Windows-Runtime, and typically are published to the Microsoft Store

- Protected Processes Light (PPL) - These processes (available from Windows 8.1) extended the protection mechanism from Vista by adding several levels of protection and even allowing third-party services to run as PPL, protecting them from intrusive access, and from termination, even by admin-level processes.

- Minimal Processes - These processes available from Windows 10 version 1607, is a truly new form of process. The address space of a minimal process does not contain the usual images and data structures that a normal process does

- Pico Processes - These processes are minimal processes with one addition: a Pico provider, which is a kernel driver that intercepts Linux system calls and translates them to equivalent Windows system calls. These processes are used with the Windows Subsystem for Linux (WSL), available from Windows 10 version 1607
[^1]

==**Process Creation**== process creation involves many internal steps done by the kernel , NTDLL , and other entities of the OS , list of steps described below:

First, the kernel opens the image (executable) file and verifies that it’s in the proper format known as Portable Executable (PE). The file extension does not matter, by the way- only the actual content does. Assuming the various headers are valid, the kernel then creates a new process kernel object and a thread kernel object, because a normal process is created with one thread that eventually should execute the main entry point.

At this point, the kernel maps the image to the address space of the new process, as well as NtDll.Dll. NtDll is mapped to every process (except Minimal and Pico processes), since it has very important duties in the final stage of process creation (outlined below) as well being the trampoline from which system calls are invoked. The final major step which is still carried out by the creator process is notifying the Windows subsystem process (Csrss.exe) of the fact that a new process and thread have been created. (Csrss can be thought of as a helper to the kernel for managing some aspects of Windows subsystem processes).

The new process is not yet ready to execute its initial code. The second part of process initialization must be carried out in the context of the new process, by the newly created thread. The star of this part is NtDll, as there is no other OS level code in the process at this time. NtDll has several duties at this point.

First, it creates the user-mode management object for the process, known as the Process
Environment Block (PEB) and the user mode management object for the first thread
called Thread Environment Block (TEB).

Then some other initializations are carried out, including the creation of the default process heap (see chapter 13), creation and initialization of the default process thread pool (chapter 9) and more. For full details, consult the Windows Internals book.

The last major part before the entry point can start execution is to load required DLLs. This part of NtDll is often referred to as the Loader. The loader looks at the import section of the executable, which includes all the libraries the executable depends upon. These typically include the Windows subsystem DLLs such as kernel32.dll, user32.dll, gdi32.dll and advapi32.dll.
[^2]

`Process creation - diagram 1`
![[WSP1/Chapter 3/d1.png]]

==**Main Function**== Once all required DLLs have been loaded and initialized successfully, control transfers to the main entry point of the executable. The entry point in question is not the actual main function provided by the developer. Instead, it’s a function provided by the C/C++ runtime, set appropriately by the linker.

There are 4 main functions that the linker can run:
```
main (mainCRTStartup) - Console application using ASCII characters
wmain (wmainCRTStartup) - Console application using Unicode characters
WinMain (WinMainCRTStartup) - GUI application using ASCII characters
wWinMain (wWinMainCRTStartup) - GUI application using Unicode characters
```

With the classic main/wmain functions, the command line arguments are broken down by the C/C++ runtime prior to calling (w)main. argc indicates the number of command-line arguments and is at least one, as the first “argument” is the full path of the executable. argv is an array of pointers to the parsed (split based on whitespace) arguments. This means argv\[0] points to the full executable path.

With the w(WinMain) functions, the parameters are as follows:
- `hInstance` represents the executable module itself within the process address space. Technically, it’s the address to which the executable is mapped.
- `hPrevInstance` should represent the HINSTANCE of a previous instance of the same executable. This value is always NULL
- `commandLine` is the command line string excluding the executable path- it’s the rest of the command line (if any). It’s not “parsed” into separate tokens- it’s just a single string

```
CommandLineToArgvW() - The function takes the command line and splits it into tokens, returning a pointer to an array of string pointers
GetCommandLine() - Get command line
```

==**Environment Values**== A console application can get the process environment variables with a third argument to main or wmain
```
int main(int argc, char* argv[], const char* env[]); // const is optional int wmain(int argc, wchar_t* argv[], const wchar_t* env[]); // const is optional
```

env is an array of string pointers, where the last pointer is NULL, signaling the end of the array. Each string is built in the following format (name = value)

`Environment variable editor - diagram 2`
![[WSP1/Chapter 3/d2.png]]

```
GetEnvironmentStrings() - to get a pointer to an environment variables memory block
SetEnvironmentVariable() - set Env variable to some value
GetEnvironmentVariable() - get Env variable value
```
 
==**Creating Processes**== Processes are created under the same user account with CreateProcess. Extended functions exist, such as CreateProcessAsUser (CreateProcess expects a valid PE file)[^3][^4]

```
CreateProcess() - Create a process, If successful, the real returned information is available via the last argument of type PROCESS_INFORMATION
```

`pProcessAttributes` and `pThreadAttributes` These two parameters are SECURITY_ATTRIBUTES pointers (for the newly created process and thread), 

`pStartupInfo` This parameter points to one of two structures, STARTUPINFO or STARTUPINFOEX. The STARTUPINFO and STARTUPINFOEX structures provide more customization options for process creation. 

`Process Creation Flag - diagram 3 & 4& 5` 
![[WSP1/Chapter 3/d3.png]]
![[WSP1/Chapter 3/d4.png]]

`STARTUPINFO structure & dwFlags - diagram 6 & 7 &8`
![[d5.png]]
![[d6.png]]

STARRTUPINFOEX contains `pAttributeList` which is a pointer to a `PROC_THREAD_ATTRIBUTE_LIST` which is used to set some special Attributes of the process 

`pAttributeList flags - diagram 9`
![[d7.png]]

```
InitializeProcThreadAttributeList() - create thread attribute list
UpdateProcThreadAttribute() - set flags in the thread attribute list
```

```
GetExitCodeProcess() - get process exit code
TerminateProcess() - terminate all threads in process
```

#### Functions discussed in this chapter 
```
GetCommandLine()
CommandLineToArgvW()

GetProcessTimes()

GetEnvironmentStrings() 
SetEnvironmentVariable() 
GetEnvironmentVariable() 

CreateProcess()
TerminateProcess()
GetExitCodeProcess()

InitializeProcThreadAttributeList()
UpdateProcThreadAttribute()
```
---
[^1]: UWP processes are special in the sense that they are suspended unwillingly when they move into the background such as when the application’s window is minimized, it suspends its self temporarily , saving resources

[^2]: Once the DLL is found, it’s loaded and its DllMain function (if exists) is called with the reason DLL_PROCESS_ATTACH indicating the DLL has now been loaded into a process.

[^3]: ShellExecuteEx. This function accepts any file, and if does not end with “EXE”, will search the registry based on the file extension to locate the associated program to execute.

[^4]:      1. The directory of the caller’s executable 
	2. The current directory of the process (discussed later in this section) 
	3. The System directory returned by GetSystemDirectory
	4. The Windows directory returned by GetWindowsDirectory 
	5. The directories listed in the PATH environment variable
