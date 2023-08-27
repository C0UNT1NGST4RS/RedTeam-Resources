Jump onto the Attacker Windows VM and open the **CreateRemoteThread** project in Visual Studio.  We're going to insert some simple Console.WriteLine's to print the PID of our injector itself, the handle information to the target process, and the base address returned by VirtualAllocEx.  This will be a useful reference for what's to come.

For instance:

```c++
// Open handle to process
var target = Process.GetProcessesByName("notepad")[0];
Console.WriteLine("Target PID: {0}", target.Id);
Console.WriteLine("Target Handle: 0x{0:X}", target.Handle.ToInt64());
```
  

Open a **Command Prompt** and set the working directory to `C:\Tools\EDR`.  Run `drvLoader.exe -i` to load the EDR driver and start the ETW tracing session.

```powershell
C:\Tools\EDR>drvLoader.exe -i
Installing driver...
Driver installed!
Starting tracing session...
```

  

This driver will automatically inject `edr_x64.dll` (the EDR DLL) into every process started from this point.  Start **notepad** (or whatever your target process will be), then run the **CreateRemoteThread** injector.

```powershell
C:\Users\Administrator>source\repos\ProcessInjection\CreateRemoteThread\bin\Debug\CreateRemoteThread.exe
My PID: 5792
Target PID: 5920
Target Handle: 0x280
Base Address: 0x14B9CBF0000
```

The EDR will hook several injection APIs and logs their use via ETW.  The loader app receives those events and prints them to the console for visibility.  The EDR DLL does not block or modify the API calls, it will simply log them.  Once running, you will see a *lot* of output, but it is possible to identify our "malicious" calls.

EDRs are likely to hook at the lowest level possible.  So instead of hooking OpenProcess in kernel32, they'll hook NtOpenProcess in ntdll.  There are a few good reasons for doing this.  The first is that OpenProcess will itself call NtOpenProcess, and hooking one API is easier than hooking two.  The second is that other (notorious) APIs like MiniDumpWriteDump call NtOpenProcess in the background (to open a handle to LSASS).  This allows the EDR to catch other suspicious activity somewhat indirectly.

```shell
[PID: 5792] Arch: x64, CommandLine: 'C:\Users\Administrator\source\repos\ProcessInjection\CreateRemoteThread\bin\Debug\CreateRemoteThread.exe'
[PID: 5792] NtAllocateVirtualMemory(0xFFFFFFFF, 4096, 0x1000, 0x04)
```

Use `Ctrl + C` to cancel the ETW logging and `drvLoader.exe -u` to uninstall the driver.

For brevity, the API call of interest are as follows:
```shell
[PID: 5792] NtOpenProcess(5920, 2035711)
[PID: 5792] NtAllocateVirtualMemory(0x280, 263168, 0x3000, 0x04)
[PID: 5792] NtWriteVirtualMemory(0x280, 0x9CBF0000, 263168)
[PID: 5792] NtCreateThreadEx(0x9CBF0000)
```

`[PID: 5792]` is the source of the ETW log - we know our CreateRemoteThread injector had a PID of 5792 from its console output.  In the call to NtOpenProcess, the target PID is 5920 - the PID of the target instance of notepad.  `2035711` is an integer representing the level of privilege being requested for the handle.  In hex, this is **1F0FFF**.  If we cross-reference this with the [OpenProcess](http://pinvoke.net/default.aspx/kernel32/OpenProcess.html) documentation on pinvoke.net, we see this is the value they provide for requesting "All" access.

Next is the call to NtAllocateVirtualMemory.  `0x280` is the handle to notepad returned by NtOpenProcess, and `263168` is the requested size in bytes.  If we go to the team server VM, we can confirm that this is the exact size of the Beacon shellcode.

```shell
# ls -l /root/beacon.bin 
-rw-r--r-- 1 root root 263168 Dec 20 12:23 /root/beacon.bin
```

`0x3000` is the memory allocation type (MEM_COMMIT and MEM_RESERVE), and `0x04` is the memory protection (PAGE_READWRITE).  We know this to be the case from our CreateRemoteThread injector source code.  NtProtectVirtualMemory is not hooked, but if it was, we'd also see the flip to RX.

In the call to NtWriteVirtualMemory we see `0x9CBF0000`, which is the base address returned to our injector by NtAllocateVirtualMemory.  This is confirmed on its console output (minus the `0x14B` prefix).  And again, we see the number of bytes written exactly match the Beacon shellcode size.  The EDR also has access to the full buffer being provided (i.e. the entire shellcode).

Finally, in the call to NtCreateThreadEx, we've logged its start address.  This matches the memory location where the shellcode has been written to.

Hopefully, this has provided some insight into how an EDR can collect information about API calls and correlate them to identify malicious intent.  It's definitely not an exact science, but can be effective.