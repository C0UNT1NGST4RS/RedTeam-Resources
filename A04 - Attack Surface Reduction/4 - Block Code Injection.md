_"This rule blocks code injection attempts from Office apps into other processes."_

This was another interesting study - now that we have a way to call the APIs under ASR, we can try and inject into other arbitrary processes or the process that we create.  Going straight to the basic VirtualAllocEx/WriteProcessMemory/CreateRemoteThread pattern, no shellcode was executed.  But looking at the memory regions of the target process showed that the memory region **was** created and the shellcode **was** sitting in memory.  The only call that failed was CreateRemoteThread.

Based on this I assume the API itself is not being blocked and that there's some other means by which ASR is preventing the creation of the thread.  Instead, we can try other injection methods that don't require the CRT API - such as QueueUserAPC.

An important note about using some C# features with G2JS.  Inline declarations don't work, so instead of things like this:

```c++
WriteProcessMemory(
    pi.hProcess,
    baseAddress,
    shellcode,
    shellcode.Length,
    out _);
```

  

You need to do:

```csharp
IntPtr bytesWriten;
WriteProcessMemory(
    pi.hProcess,
    baseAddress,
    shellcode,
    shellcode.Length,
    out bytesWriten);
```
  
> **EXERCISE**  
Write an injector that works with G2JS to spawn a process and inject shellcode into it.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/win32calc-inject.png)

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/win32calc-beacon.png)