**CreateRemoteThread** behaves in much the same way as CreateThread, but allows you to start a thread in a process other than your own.  This can be used to inject shellcode into a different process.  The injection steps are practically identical to the previous example, but we use slightly different APIs.

-   VirtualAlloc -> VirtualAllocEx
-   Marshal.Copy -> WriteProcessMemory
-   VirtualProtect -> VirtualProtectEx
-   CreateThread -> CreateRemoteThread

Manually open a process such as notepad and use its PID as your target process.  To open a handle to the target process, you can use the OpenProcess API or the .NET Process class.

```csharp
using System.Diagnostics;

namespace CreateRemoteThread
{
    internal class Program
    {
        static void Main(string[] args)
        {
            var process = Process.GetProcessById(1234);
        }
    }
}
```

  

For convenience, you may also pass the PID on the command line.

```csharp
var pid = int.Parse(args[0]);
var process = Process.GetProcessById(pid);
```

  

The process variable will have a property called `Handle` which can be passed to the APIs.  The rest is the same pattern, and you should get a Beacon running in your target process.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/process-injection/createremotethread/beacon.png)