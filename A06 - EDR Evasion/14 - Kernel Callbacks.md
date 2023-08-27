Windows drivers are able to register callback routines in the kernel, which are triggered when particular events occur.  These can include process and thread creation, image loads and registry operations.  For example, when a user attempts to launches an exe, a notification is sent to any driver that has registered a callback with [PsSetCreateProcessNotifyRoutineEx](https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/ntddk/nf-ntddk-pssetcreateprocessnotifyroutineex).  The driver then has an opportunity to take some action, such as blocking the process from starting, or injecting a userland DLL into it.

These callbacks are stored within the kernel and each routine has its own "array", such as the PspCreateProcessNotifyRoutine array.  Each entry in the array contains a pointer to a function within the driver that has registered the callback.  When an event in question occurs, the kernel iterates over each entry in the array and executes the callback function within each driver.  Applications such as Sysmon have a driver component which register these callbacks, which is how they collect some of their telemetry.  AV and EDRs do as well.

Because these callbacks are stored inside kernel memory, there is scope to remove or modify them if we have an arbitrary read/write primitive.  The most typical way to achieve this is via your own driver - either one that you can load yourself (local admin to kernel is not a security boundary) or by exploiting a vulnerable driver that's already installed.

Since this is not a driver exploitation course, we'll focus on the former method.  There are several resources that we can lean on including the [Mimikatz driver](https://github.com/gentilkiwi/mimikatz/tree/master/mimidrv), but the most accomplished in this space is [fdiskyou](https://twitter.com/fdiskyou)'s [Windows Ps Callback Experiments](https://github.com/houseofxyz/windows-ps-callbacks-experiments).

```powershell
C:\Users\Administrator>sc create evilDriver type= kernel binPath= C:\Tools\evil-driver\evil.sys
[SC] CreateService SUCCESS

C:\Users\Administrator>sc start evilDriver

SERVICE_NAME: evilDriver
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
        PID                : 0
        FLAGS              :
```

The driver exposes several IOCTLs that we can interact with from userland, via the `evil.exe` tool.  `-l` will list all process, thread and image load callbacks.


![](https://rto2-assets.s3.eu-west-2.amazonaws.com/kernel/list-callbacks.png)  

### CreateProcessNotify

Highlighted above are the callbacks registered by the Sysmon driver -  the process callback is used to log EID1 ProcessCreate events.  You may open **Event Viewer > Applications and Service Logs > Microsoft > Windows > Sysmon > Operational** to see any that are logged.

	Click on "**Filter Current Log**" in the Actions menu and put `1` in place of `<All Event IDs>` and click OK.

To temporarily disable the process callback, we can overwrite the instructions with a `RET`.  This means that when the sysmon callback routine is executed, it will immediately return, thus not doing anything.

```powershell
C:\Tools\evil-driver>evil.exe -pp 5
Patching index: 5 with a RET (0xc3)
``` 

Now we can launch as many processes as we like, and they won't appear in the Event Viewer.

	Clear the current log to make it easier to verify that no new ones appear.


Restore the original callback routine with `-rp` and ProcessCreate events will start appearing again.

```powershell
C:\Tools\evil-driver>evil.exe -rp 5
Rolling back patched index: 5 to the original values
```

## CreateThreadNotify

This notification is triggered any time an application creates a new thread - either a new thread inside itself (CreateThread) or a thread in another process (CreateRemoteThread).  This is obviously useful for detecting process injection techniques.

The EDR driver will output kernel debug messages which you can see with DebugView (found at `C:\Tools\dbgview64.exe`).  Make sure kernel capture (**Capture > Kernel Capture**) is enabled.

Running the CreateRemoteThread or even the NtCreateThreadEx version using D/Invoke syscalls, we will see the driver getting a notification of the new thread.  This is a great example of where the use of syscalls will not help avoid detection.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/kernel/kernel-capture.png)

  

As above, our evil driver can be used to disable the callback.  This time, let's delete it entirely with the `-dt` option.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/kernel/remove-thread-callback.png)

  

The evil driver also prints debug messages, you should see `[evilDrv] Callback Removed!` in DebugView.  Launch the injector again and the driver won't print the thread creation message.