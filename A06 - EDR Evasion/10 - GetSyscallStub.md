In addition to manual mapping, D/Invoke has a method called **GetSyscallStub.**

```csharp
var ntOpenProcessPtr = Generic.GetSyscallStub("NtOpenProcess");
```
  

This works by reading the original syscall stub from ntdll on disk and copying it into a new memory region within our calling process.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/syscalls/getsyscallstub.png)

  

The method returns a function pointer (the base address of the new memory region), which we can marshal to a delegate using **Marshal.GetDelegateForFunctionPointer**.

```csharp
var ntOpenProcess = Marshal.GetDelegateForFunctionPointer(ntOpenProcessPtr, typeof(Native.DELEGATES.NtOpenProcess)) as Native.DELEGATES.NtOpenProcess;
```


Then simply call the function.

```csharp
var oa = new Data.Native.OBJECT_ATTRIBUTES();
var cid = new Data.Native.CLIENT_ID
{
    UniqueProcess = (IntPtr)target.Id
};

var hProcess = IntPtr.Zero;

var status = (uint)ntOpenProcess(
    ref hProcess,
    Data.Win32.Kernel32.ProcessAccessFlags.PROCESS_ALL_ACCESS,
    ref oa,
    ref cid);
``` 

If we're writing this inside a tool that will close after the syscalls have been executed, we don't need to worry too much about this allocated memory.  If implementing it inside a tool that will remain running (e.g. a C2 implant), then we should free the memory region using **NtFreeVirtualMemory** to avoid leaving stubs in memory that could be used as an IoC of this technique.