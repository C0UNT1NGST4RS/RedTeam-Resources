Cobalt Strike's `spawnto` value controls which binary is used as a temporary process for various post-exploitation workflows.  Execute this PowerShell one-liner using `powerpick` and keep an eye on Process Hacker.

```
beacon> powerpick Start-Sleep -s 30
``` 

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/spawnto/rundll32.png)

  

We'll see that rundll32 is spawned as a child of the current Beacon process (in this case, PowerShell).  This is Cobalt Strike's default spawnto and is highly monitored as arbitrary processes spawning rundll32 is not a normal occurrence.  In fact, you will probably find that Defender kills your Beacon (note the PID correlation).  This is a behavioural detection and is not circumvented using AMSI bypasses in Beacon.
  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/spawnto/rundll32-blocked.png)

  

You can change the spawnto binary for an individual Beacon during runtime with the `spawnto` command.  There is a separate configuration for x86 and x64.

```
beacon> help spawnto
Use: spawnto [x86|x64] [c:\path\to\whatever.exe]
```

Sets the executable Beacon spawns x86 and x64 shellcode into. You must specify a
full-path. Environment variables are OK (e.g., `%windir%\sysnative\rundll32.exe`)

Do not reference `%windir%\system32\ directly`. This path is different depending
on whether or not Beacon is x86 or x64. Use `%windir%\sysnative`\ and 
`%windir%\syswow64\` instead.

Beacon will map `%windir%\syswow64\` to system32 when WOW64 is not present.

  

Obviously, the idea is to pick something that would not be out of place for the current Beacon context.  For demonstration purposes, let's spawn a new Beacon and change it to Notepad.

```
beacon> spawnto x64 %windir%\sysnative\notepad.exe
beacon> powerpick Start-Sleep -s 30
```


![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/spawnto/notepad.png)

  

This time, the Beacon survives.  There is nothing special happening here - Beacon uses the CreateProcess API to start whatever spawnto it's configured with.  If you want to set the default spawnto in your C2 profile, you can do so in the post-ex block.

post-ex {
    set spawnto_x86 "%windir%\\syswow64\\notepad.exe";
    set spawnto_x64 "%windir%\\sysnative\\notepad.exe";
}