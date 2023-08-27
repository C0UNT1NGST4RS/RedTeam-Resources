Microsoft SQL Server is a relational database management system commonly found in Windows environments. They’re typically used to store information to support a myriad of business functions. In addition to the obvious data theft opportunities, they also have a large attack surface, allowing code execution, privilege escalation, lateral movement and persistence.

[PowerUpSQL](https://github.com/NetSPI/PowerUpSQL) is an excellent tool for enumerating and interacting with MS SQL Servers.

There are a few "discovery" cmdlets available for finding MS SQL Servers, including `Get-SQLInstanceDomain`, `Get-SQLInstanceBroadcast` and `Get-SQLInstanceScanUDP`.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> powershell Get-SQLInstanceDomain

ComputerName     : srv-1.dev.cyberbotic.io
Instance         : srv-1.dev.cyberbotic.io,1433
DomainAccountSid : 15000005210002361191261941702819312113313089172110400
DomainAccount    : svc_mssql
DomainAccountCn  : MS SQL Service
Service          : MSSQLSvc
Spn              : MSSQLSvc/srv-1.dev.cyberbotic.io:1433
LastLogon        : 5/14/2021 2:24 PM
Description      : 
```

`Get-SQLInstanceDomain` works by searching for SPNs that begin with `MSSQL*`. This output shows that `srv-1.dev.cyberbotic.io` is running an instance of MS SQL server, being run under the context of the `svc_mssql` domain account.

BloodHound also has an edge for finding potential MS SQL Admins, based on the assumption that the account running the SQL Service is also a sysadmin (which is very common);

```
MATCH p=(u:User)-[:SQLAdmin]->(c:Computer) RETURN p
```

You may also search the domain for groups that sound like they may have access to database instances (for instance, there is a **MS SQL Admins** group).

Once you've gained access to a target user, `Get-SQLConnectionTest` can be used to test whether or not we can connect to the database.

```
beacon> powershell Get-SQLConnectionTest -Instance "srv-1.dev.cyberbotic.io,1433" | fl

ComputerName : srv-1.dev.cyberbotic.io
Instance     : srv-1.dev.cyberbotic.io,1433
Status       : Accessible
```

Then use `Get-SQLServerInfo` to gather more information about the instance.

```
beacon> powershell Get-SQLServerInfo -Instance "srv-1.dev.cyberbotic.io,1433"

ComputerName           : srv-1.dev.cyberbotic.io
Instance               : SRV-1
DomainName             : DEV
ServiceProcessID       : 3960
ServiceName            : MSSQLSERVER
ServiceAccount         : DEV\svc_mssql
AuthenticationMode     : Windows Authentication
ForcedEncryption       : 0
Clustered              : No
SQLServerVersionNumber : 13.0.5026.0
SQLServerMajorVersion  : 2016
SQLServerEdition       : Standard Edition (64-bit)
SQLServerServicePack   : SP2
OSArchitecture         : X64
OsMachineType          : ServerNT
OSVersionName          : Windows Server 2016 Datacenter
OsVersionNumber        : SQL
Currentlogin           : DEV\bfarmer
IsSysadmin             : Yes
ActiveSessions         : 1
```

>  If there are multiple SQL Servers available, you can chain these commands together to automate the data collection.  
```
beacon> powershell Get-SQLInstanceDomain | Get-SQLConnectionTest | ? { $_.Status -eq "Accessible" } | Get-SQLServerInfo
```  

From this output, we can see that bfarmer has the **sysadmin** role on the instance. There are several options for issuing queries against a SQL instance.

`Get-SQLQuery` from PowerUpSQL:

```
beacon> powershell Get-SQLQuery -Instance "srv-1.dev.cyberbotic.io,1433" -Query "select @@servername"

Column1
-------
SRV-1 
```

  

`mssqlclient.py` from Impacket via proxychains:

```
root@kali:~# proxychains python3 /usr/local/bin/mssqlclient.py -windows-auth DEV/bfarmer@10.10.17.25
ProxyChains-3.1 (http://proxychains.sf.net)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:1433-<><>-OK
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(SRV-1): Line 1: Changed database context to 'master'.
[*] INFO(SRV-1): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (130 19162)
[<>] Press help for extra shell commands
SQL> select @@servername;

SRV-1
```

  

Or a Windows SQL GUI (such as [HeidiSQL](https://www.heidisql.com/) or even [SMSS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms)) via Proxifier:

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/heidi.png)

  

Let's have a look at some ways in which we can abuse access to this SQL instance.




Pwn3rzs - CyberArsenal