Session Prepping is a term to describe how you can "prep" your session after landing an initial Beacon on a machine, which is an important step after initial compromise or lateral movement.  Your strategies and TTPs should be defined upfront based on the threat you're emulating, but for the sake of this section I'm going to provide the following:

Send a weaponised Word document in a phishing email that will locate a running instance of Edge, IE or Chrome, then inject a Beacon payload into it.  If an instance is not found, spawn one.  Once the Beacon has landed, set the spawnto to whichever browser binary we landed in (or spawned).  Keep the PPID set to the Beacon process.

Rationale:

-   Browsers make outbound HTTP/S connections by design.
-   Edge, IE and Chrome are the most popular (you could include Firefox as well).
-   Browsers legitimately spawn new child processes per tab.

Beacon is running in msedge.exe, PID 1048.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/msedge-beacon.png)

On the target, the process tree looks like this:

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/msedge-ph.png)

Of the 7 child processes there, one of them is a Beacon post-ex capability running, the others are legitimate tabs.

---

With lateral movement, we often don't get much of a choice in what we're executing on the target, particularly when using the default `jump` commands.  For example, `jump winrm64` will land us in a PowerShell process and `jump psexec64` in the default spawnto of the C2 profile.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/rundll32-beacon.png)

  

After jumping with psexec, the Beacon is living in rundll32.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/rundll32-ph.png)

  

rundll32.dll is a major outlier here because it's running in Session ID 0, but not PPID'd to an existing service executable.  This would be true for whatever spawnto we were using, as this is just the nature of how Cobalt Strike's psexec implementation works.  The service and service executable are cleaned up and removed after execution, which leaves the process hosting the payload in this orphaned state.  We don't want to continue operating in this Beacon, so we should prep the session before performing any post-ex actions on this host.

There are two paths we can take from here depending on what we want to achieve.

If the RTO2\amoss user is the target, we can move into their desktop session and live entirely in their user space.  We would be dropping down from high-integrity to medium, but that might not even matter.  The easiest method of doing this is to inject a payload into one of the user's processes (there aren't many here, so let's just use explorer).

```
beacon> inject 4628 x64 smb
[+] established link to child beacon: 10.10.120.75
```

The Beacon chain is now going like this:

------------------------      ------------------------       -------------------------
|  cbridges @ WKSTN-1  |      |   SYSTEM @ WKSTN-2    |      |    amoss @ WKSTN-2    |
|   msedge.exe (1048)  |  =>  |  rundll32.dll (5844)  |  =>  |  explorer.exe (4628)  | 
------------------------      ------------------------       -------------------------

So next, we need to `exit` the SYSTEM session and `link` to the session running as amoss from cbridges.  We can leave the PPID since it's ok for processes to be a child of explorer, and then set the spawnto to something the user might execute.

beacon> spawnto x64 %windir%\\sysnative\\notepad.exe
beacon> spawnto x86 %windir%\\syswow64\\notepad.exe

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/spawnto-notepad.png)

  

The other option, is if we want to maintain our SYSTEM level access in Session 0.  The strategy that makes the most sense to me is to hide ourselves as a child of an existing service executable.  Most of these are found as children of **services.exe** (the "Services and Controller app") and the vast majority of default Windows services run as **svchost.exe**.  Many of these will also spawn their own children such as SearchApp.exe, taskhostw.exe, ctfmon.exe and others.

The "problem" with these core Windows processes is that they are protected, so you can't arbitrarily open handles to them, even as SYSTEM.

```shell
beacon> getuid
[*] You are NT AUTHORITY\SYSTEM (admin)

beacon> inject 840 x64 smb
[-] could not open process 840: 5
[-] could not connect to pipe
```

  

A more reliable bet is to find a third-party service, because they will often have a lower level of protection.  Let's have a look at the Amazon SSM Agent as an example.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/ssm-agent.png)

  

**amazon-ssm-agent.exe** is the service executable for the AmazonSSMAgent service.

```shell
SERVICE_NAME: AmazonSSMAgent
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Amazon SSM Agent
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

  

This service spawns **ssm-agent-worker.exe** as children.  This could be a good candidate - we could inject into the agent process and set our spawnto to the worker executable.

```
beacon> inject 2796 x64 smb
[+] established link to child beacon: 10.10.120.75

beacon> spawnto x64 %ProgramFiles%\Amazon\SSM\ssm-agent-worker.exe
```
  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/session-prep/agent-worker.png)

  

These are just some examples based on the situation you find yourself in.  You should be willing to enumerate a host and adapt to blend into what looks "normal".