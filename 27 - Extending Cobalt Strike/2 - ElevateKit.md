The [Elevate Kit](https://github.com/Cobalt-Strike/ElevateKit) provides a means of extending the `elevate` command with third-party privilege escalation and UAC bypass techniques.  Beacon has 2 built-in elevate commands.

```
beacon> elevate

Beacon Local Exploits
=====================

    Exploit                         Description
    -------                         -----------
    svc-exe                         Get SYSTEM via an executable run as a service
    uac-token-duplication           Bypass UAC with Token Duplication
```
  

After we've loaded `elevate.cna`, that shoots up to 7.

```
beacon> elevate

Beacon Local Exploits
=====================

    Exploit                         Description
    -------                         -----------
    cve-2020-0796                   SMBv3 Compression Buffer Overflow (SMBGhost) (CVE 2020-0796)
    ms14-058                        TrackPopupMenu Win32k NULL Pointer Dereference (CVE-2014-4113)
    ms15-051                        Windows ClientCopyImage Win32k Exploit (CVE 2015-1701)
    ms16-016                        mrxdav.sys WebDav Local Privilege Escalation (CVE 2016-0051)
    svc-exe                         Get SYSTEM via an executable run as a service
    uac-schtasks                    Bypass UAC with schtasks.exe (via SilentCleanup)>
    uac-token-duplication           Bypass UAC with Token Duplication
```

Let's inspect `elevate.cna` to see how it works (it's actually very simple).

Each "technique" is implemented within a `sub` block, such as `sub schtasks_elevator { }`. The `sub` keyword defines a function, within which is the code to execute.

Underneath each function is a `beacon_elevator_register` line, which is defined [here](https://www.cobaltstrike.com/aggressor-script/functions.html#beacon_elevator_register). It takes 3 parameters:

-   the exploit short name
-   the exploit description
-   the function that implements the exploit

The name of the `sub` function itself does not matter (although it's good practice to call it something meaningful), but it must match the function name provided to `beacon_elevator_register`.

Now let's review the actual code inside `sub schtasks_elevator`. Raphael's comments are very good, so most of it should be clear.

The first declaration, **local**, defines variables that are local to the current function. So once `schtasks_elevator` has executed, these variables disappear. Sleep can have `global`, `closure-specific` and `local` scopes. More information on scopes can be found in **5.2 Scalar Scope** of the Sleep manual.

**[btask](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#btask)** reports that Beacon has been tasked. This outputs to the console in the form of `[*] Tasked beacon to ...` and also adds to the CS data model used in the automatic report generation. The Beacon ID is (almost always) passed in as `$1`.

The next line has a few embedded elements that will open a handle to read a file. [openf](http://sleep.dashnine.org/manual/openf.html) and [getFileProper](http://sleep.dashnine.org/manual/getFileProper.html) are Sleep functions and [script_resource](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#script_resource) is Aggressor. All this returns a path to `Invoke-EnvBypass.ps1` in the `ElevateKit\modules` folder.

Once the `Invoke-EnvBypass.ps1` file has been read, **[beacon_host_script](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#beacon_host_script)** uploads it to Beacon's little internal webserver and returns a short oneliner to invoke it.

**[transform](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#transform)** takes in Beacon shellcode and outputs it in a different format (in this case base64 for use with PowerShell). Where `$1` is the Beacon ID, `$2` is the name of the listener provided in the UI.

Finally, **[powerpick](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#bpowerpick)** is used to execute it all.

The flexibility of Beacon means that we can leverage anything from PowerShell, execute-assembly, shellcode injection, DLL injection and more.

```
beacon> elevate uac-schtasks tcp-local
[*] Tasked Beacon to run windows/beacon_bind_tcp (127.0.0.1:4444) in a high integrity context
[+] host called home, sent: 352789 bytes
[+] established link to child beacon: 10.10.5.110
```