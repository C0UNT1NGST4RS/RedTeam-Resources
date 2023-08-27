[SysWhispers](https://github.com/jthuraisamy/SysWhispers) (and its successor, [SysWhispers2](https://github.com/jthuraisamy/SysWhispers2)) can generate header and assembly files to facilitate making syscalls from native code.  The primary difference between the two is that SysWhispers relies on static syscall tables, and SysWhispers2 finds the SSNs at runtime.  The main advantage of the later is that you don't have to be concerned with getting the correct SSNs for a particular Windows version or having to recompile your tools all the time.

A clone of SysWhispers2 can be found in `/home/ubuntu/SysWhispers2`.  As an example, let's use the "common" present to generate x64 syscalls and output them in "masm" format.

```shelll
ubuntu@teamserver ~/SysWhispers2 (main)> python3 syswhispers.py -p common -a x64 -l masm -o syscalls

                  .                         ,--.
,-. . . ,-. . , , |-. o ,-. ,-. ,-. ,-. ,-.    /
`-. | | `-. |/|/  | | | `-. | | |-' |   `-. ,-'
`-' `-| `-' ' '   ' ' ' `-' |-' `-' '   `-' `---
     /|                     |  @Jackson_T
    `-'                     '  @modexpblog, 2021

SysWhispers2: Why call the kernel when you can whisper?

Common functions selected.

Complete! Files written to:
        syscalls.h
        syscalls.c
        syscallsstubs.x64.asm
```
  

Before we can use these, open Visual Studio and create a new C++ Console App.  Then copy `syscalls.h`, `syscall.c` and `syscallsstubs.x64.asm` to the root of the project.

```powershell
C:\Users\Administrator>pscp -i Desktop\ssh.ppk ubuntu@10.10.0.69:/home/ubuntu/SysWhispers2/syscalls* C:\Tools\SysWhispersDemo\.
syscallsstubs.x64.asm     | 7 kB |   7.3 kB/s | ETA: 00:00:00 | 100%
syscalls.h                | 12 kB |  12.7 kB/s | ETA: 00:00:00 | 100%
syscalls.c                | 4 kB |   4.0 kB/s | ETA: 00:00:00 | 100%
```

In the Solution Explorer, click the **Show All Files** button which will reveal the files.  Highlight each one, right-click and select **Include in Project**.  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/syscalls/syswhispers/solution-explorer.png)


Next, click the **Project** menu button at the top of the Visual Studio window and select **Build Customizations**.  Tick the **masm** checkbox and click **OK**.  Right-click on the `syscallstubs.x64.asm` file in the Solution Explorer and choose Properties.  Change the **Item Type** to **Microsoft Macro Assembler** and click **OK**.

Finally, add `#include "syscalls.h"` to your main .cpp file and the syscalls will be available for use.  For example:

```cpp
#include <Windows.h>
#include <iostream>
#include "syscalls.h"

int main(int argc, const char* argv[])
{
	DWORD targetPid = atoi(argv[1]);

	OBJECT_ATTRIBUTES oa;
	InitializeObjectAttributes(&oa, 0, 0, 0, 0);

	CLIENT_ID cid = { (HANDLE)targetPid, NULL };

	HANDLE hProcess;
	NTSTATUS status = NtOpenProcess(&hProcess, PROCESS_ALL_ACCESS, &oa, &cid);

	if (status == 0)
	{
		printf("hProcess: 0x%p", hProcess);
		return EXIT_SUCCESS;
	}
	else
	{
		printf("Failed to open handle to PID %d", targetPid);
		return EXIT_FAILURE;
	}
}
```

```powershell
C:\>tasklist /FI "IMAGENAME eq explorer.exe"

Image Name                     PID Session Name        Session#    Mem Usage
========================= ======== ================ =========== ============
explorer.exe                  2900 RDP-Tcp#0                  2    138,980 K

C:\>C:\Tools\SysWhispersDemo\x64\Debug\SysWhispersDemo.exe 2900
hProcess: 0x00000000000000AC
```
