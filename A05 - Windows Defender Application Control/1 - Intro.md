WDAC is a Windows technology designed to control which drivers and applications are allowed to run on a machine.  It sounds a lot like AppLocker, but with a few key differences.  The most significant of which is that Microsoft recognises WDAC as an [official security boundary](https://www.microsoft.com/en-us/msrc/windows-security-servicing-criteria).  This means that WDAC is substantially more robust and applicable bypasses are actually fixed (and a CVE often issued to the finder).

The term "WDAC bypass" is used herein, but this is disingenuous since we're never actually bypassing WDAC at a fundamental level.  Instead, we are finding weaknesses in the policy that organisations deploy.

WDAC policies are first defined in XML format - Microsoft ships several base policies which can be found under `C:\Windows\schemas\CodeIntegrity\ExamplePolicies`.  Multiple policies can be merged into a single policy, which is then packaged into a `.p7b` file and pushed out via GPO or another management platform such as Intune.  If you can find the GPO responsible for pushing that policy, you can find where it lives.

```shell
beacon> powershell-import C:\Tools\PowerSploit\Recon\PowerView.ps1
beacon> powerpick Get-DomainGPO -Name *WDAC* -Properties GpcFileSysPath

gpcfilesyspath                                                                              
--------------                                                                              
\\redteamops2.local\SysVol\redteamops2.local\Policies\{36C8DC4C-6B44-4EEB-9099-CA0E050DD6DE}

beacon> download \\redteamops2.local\SysVol\redteamops2.local\Policies\{36C8DC4C-6B44-4EEB-9099-CA0E050DD6DE}\Machine\Registry.pol
[*] started download of \\redteamops2.local\SysVol\redteamops2.local\Policies\{36C8DC4C-6B44-4EEB-9099-CA0E050DD6DE}\Machine\Registry.pol (448 bytes)
[*] download of Registry.pol is complete

PS C:\Users\Administrator\Desktop> Parse-PolFile .\Registry.pol

KeyName     : SOFTWARE\Policies\Microsoft\Windows\DeviceGuard
ValueName   : DeployConfigCIPolicy
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 1

KeyName     : SOFTWARE\Policies\Microsoft\Windows\DeviceGuard
ValueName   : ConfigCIPolicyFilePath
ValueType   : REG_SZ
ValueLength : 116
ValueData   : \\redteamops2.local\SYSVOL\redteamops2.local\CIPolicy.p7b
```

  

The **ValueData** field contains the location of the WDAC policy itself.  This is usually somewhere like a central file share, so that machines can read the policy in order to apply it.

```shell
beacon> download \\redteamops2.local\SYSVOL\redteamops2.local\CIPolicy.p7b
[*] started download of \\redteamops2.local\SYSVOL\redteamops2.local\CIPolicy.p7b (2360 bytes)
[*] download of CIPolicy.p7b is complete
```

  

>  If you have filesystem access to a machine that has the WDAC policy applied, you can grab the p7b from `C:\Windows\System32\CodeIntegrity`.

  

[Matt Graeber](https://twitter.com/mattifestation) wrote [CIPolicyParser.ps1](https://gist.github.com/mattifestation/92e545bf1ee5b68eeb71d254cec2f78e), which can reverse this binary format back into XML for easy reading.

```powershell
PS C:\Users\Administrator\Desktop> ipmo C:\Tools\CIPolicyParser.ps1
PS C:\Users\Administrator\Desktop> ConvertTo-CIPolicy -BinaryFilePath .\CIPolicy.p7b -XmlFilePath policy.xml
```

WDAC allows for very granular control when it comes to trusting an application.  The most commonly used ones include:

-   Hash - allows binaries to run based on their hash values.
-   FileName - allows binaries to run based on their original filename.
-   FilePath - allows binaries to run from specific file path locations.
-   Publisher - allows binaries to run that are signed by a particular CA.