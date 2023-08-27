Dynamic Invoke ([D/Invoke](https://github.com/TheWover/DInvoke)) is an open-source C# project intended as a direct replacement for P/Invoke.  It has a number of powerful primitives that can be combined to do some really neat things, including:

-   Invoke unmanaged code without P/Invoke.
-   Manually map unmanaged PE's into memory and call their associated entry point or an exported function.
-   Generate syscall wrappers for native APIs.

For now, we'll focus on the first point.  But one question you might have is why avoid P/Invoke?

Tools such as [pestudio](https://www.winitor.com/) can inspect a compiled .NET assembly and identify "suspicious" P/Invoke usage.  In the example below, this assembly calls OpenProcess, VirtualAllocEx, WriteProcessMemory and CreateRemoteThread.  These APIs are synonymous with process injection and would therefore raise some alarms.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/pestudio-pinvoke.png)