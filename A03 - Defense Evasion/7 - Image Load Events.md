An "image load" is when a process loads a DLL into memory.  This is a perfectly legitimate thing to happen, and all processes will have a boat-load of DLLs loaded.  Here's an example of a normal Notepad process.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/image-load/notepad-normal.png)

  

Ingesting all image load events into a SIEM is not completely viable due to the huge volume.  But defenders can selectively forward specific image loads based on known attacker TTPs.  One example is the use of `execute-assembly`.

The Cobalt Strike implementation will:

-   Spawn a temporary process (whatever is configured as the spawnto binary).
-   Load the .NET CLR (Common Language Runtime) into that process.
-   Execute the given .NET assembly in memory of that process.
-   Get the output and kill the process.

The .NET CLR (and other associated DLLs) is usually only loaded by .NET assemblies - native programs tend not to.  If your spawnto is set to a native binary (such as notepad) and you use `execute-assembly`, defenders could see that a native binary has loaded the CLR.

Here's an example Sysmon event where notepad.exe has loaded **clr.dll**.

```shell
Image loaded:
ProcessId: 696
Image: C:\Windows\System32\notepad.exe
ImageLoaded: C:\Windows\Microsoft.NET\Framework64\v4.0.30319\clr.dll
Description: Microsoft .NET Runtime Common Language Runtime - WorkStation
```

  

One way to avoid this style of detection is to set the spawnto to a .NET assembly - there are plenty that exist on Windows by default.

Image load events can also be helpful in tracking down capabilities such as Mimikatz, because it can load various DLLs that handle cryptography, and interactions with the Windows Credential Vault etc.