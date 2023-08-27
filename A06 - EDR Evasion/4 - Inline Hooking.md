At a basic level, inline (or detour/trampoline) hooking is achieved by patching the API call instructions in memory.  When the EDR's userland DLL is injected into a process, it resolves the address of each API function that it wants to hook, and patches a **jmp** instruction at the start for each one.  This will redirect the execution flow to a "detour" method inside the EDR's DLL memory space.  The EDR will inspect the API call and decide what to do with it.

If deemed malicious, the call can just be blocked.  If not, it can be forwarded to the original API function.  It may also raise an alert.

Below is an example of how the MessageBox API appears under normal circumstances in [WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools).  We can see that the first instruction is a **sub**.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/inline/messagebox-windbg.png)

However, when the API is hooked, that instruction is patched with a **jmp**.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/inline/messagebox-windbg-hooked.png)