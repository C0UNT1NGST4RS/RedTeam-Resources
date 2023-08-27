You can also tunnel Metasploit modules through Beacon's SOCKS proxy, which is really useful with remote exploits.

In Cobalt Strike, go to **View > Proxy Pivots**, highlight the existing SOCKS proxy and click the **Tunnel** button. This gives you a string which looks like: `setg Proxies socks4:10.10.5.120:1080`. Paste this into `msfconsole` and any remote modules will go via Beacon.

To stop the SOCKS proxy, use `socks stop` or **View > Proxy Pivots > Stop**.