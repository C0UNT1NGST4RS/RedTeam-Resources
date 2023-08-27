Thread Stack, or Call Stack Spoofing, is an in-memory evasion technique which aims to hide references to shellcode on a call stack.  But first - what is a call stack?  In general terms, a "stack" is a LIFO (last in, first out) collection, where data can be "pushed" (added) or "popped" (removed).

  

![](https://miro.medium.com/max/874/1*rLV0q6if8Drx1PbrncybXw.png)

Image credit:  Ryan Mariner Farney

  

The main purpose of a call stack (particularly in this context) is to keep track of where a routine should return to once it's finished executing.  For instance, the MessageBoxW API in kernel32.dll has no knowledge of anything that may call it.  Before calling this API, a return address is pushed onto the stack, so that once MessageBoxW has finished, execution flow can return back to the calling application.

Let's see what this means in the context of Beacon.  Here, I have a Beacon running on Attacker Windows using the default EXE artefact.  Process Hacker can display the running threads.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/no-spoof-threads.png)

  

Double-click (or right-click > Inspect) on the main (highlighted) thread will reveal the content of the call stack.  Here, we can see a call to **SleepEx** in `KernelBase.dll` and then two seemingly random memory addresses.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/call-stack-4360.png)

  

Cross-referencing the memory regions, we find that it leads us straight to the Beacon payload in memory.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/mapped-pe.png)

  

This is showing that after the SleepEx call has completed, execution will return to Beacon, because this is where the API was called from.  The main red flag being that the reference is directly to a memory address, rather than an exported function.  This can be picked up by both automated tooling and manual analysis.

  

Stack spoofing can be enabled via the Artifact Kit by setting the "stack spoof" option to `true`.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/build-script.png)

  

For example:  `./build.sh pipe VirtualAlloc 271360 5 true true /tmp/dist>`.  Copy the artifacts across to the Windows machine, load the CNA script and generate a new payload.  When inspecting this process inside Process Hacker, we will see that the call stack for the main thread looks a little different.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/call-stack-6416.png)

  

The direct reference to memory addresses have been replaced.

At the time of writing, this implementation hooks the Beacon sleep function and [](https://docs.microsoft.com/en-us/windows/win32/procthread/fibers)overwrites its memory with a  small [trampoline](https://en.wikipedia.org/wiki/Trampoline_(computing)).  This zeros out the return address to prevent stack walking.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/stack-spoof/x64-trampoline.png)

  

After setting up the trampoline, it uses [Fiber](https://docs.microsoft.com/en-us/windows/win32/procthread/fibers) APIs such as CreateFiber, SwitchToFiber and DeleteFiber to execute alternate units of work, like WaitForSingleObject.

The source code for achieving this is in `arsenal-kit/kits/artifact/src-common/spoof.c`.