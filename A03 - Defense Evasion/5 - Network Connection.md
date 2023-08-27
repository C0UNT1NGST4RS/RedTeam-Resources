When a process makes a network connection it can be logged in Sysmon, a network monitoring device, or seen using a local tool such as netstat.

This is an example Sysmon event for when Beacon (running in powershell.exe) performs a check-in.  We can see a connection to 10.10.5.39 (Redirector 1) is made on port 80.  In a target environment, defenders would likely see the outbound connection going to their boundary firewall or web proxy.

A new event will be generated for every check-in made by the Beacon, so if you're on `sleep 0`, get ready to be flooded.

```shell
Network connection detected:
ProcessId: 5696
Image: C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
User: RTO2\cbridges
Protocol: tcp
Initiated: true
SourceIp: 10.10.120.106
SourceHostname: wkstn-1.redteamops2.local
SourcePort: 65189
SourcePortName: -
DestinationIp: 10.10.5.39
DestinationHostname: -
DestinationPort: 80
DestinationPortName: http
```

  
There are no magic bypasses to this per se - as the operator, you should consider whether or not it makes sense for your host process to be making network connections.  An HTTP Beacon will make HTTP/S connections, the TCP Beacon TCP connections and the SMB Beacon named pipe connections.

You may consider a web browser process more appropriate for HTTP/S connections.