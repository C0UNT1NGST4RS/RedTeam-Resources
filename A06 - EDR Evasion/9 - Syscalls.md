x86 CPUs have four privilege levels, known as rings.  They range from Ring 0 (most privileged) to Ring 3 (least privileged) and control access to memory and CPU operations.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/syscalls/rings.png)

  

Windows only supports Rings 0 and 3 - referred to as "kernel" and "user mode" (or "user land") respectively.  The majority of user activity will occur in Ring 3 but applications can cross into Ring 0.  The Win32 APIs (kernel32.dll, user32.dll etc) are designed to be the first port of call for developers.  These APIs will call lower-level APIs in ntdll.dll.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/syscalls/kernel-transition.png)

  

For instance - an application may call [CreateFileW](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) in kernel32.dll to open a file.  CreateFileW will call [NtCreateFile](https://docs.microsoft.com/en-us/windows/win32/api/winternl/nf-winternl-ntcreatefile) in ntdll.dll, and ntdll.dll will use a system call (or "syscall") to transition into the kernel and access the filesystem hardware.

The syscall can be seen rather easily in a debugger.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/syscalls/syscall.png)

  

Every syscall has a known "system service number" (SSN).  NtCreateFile is **0x0055**, which is why see a **mov eax, 55h**.  You can find these from resources such as [j00ru's system call table](https://j00ru.vexillium.org/syscalls/nt/64/).  Once the CPU registers are setup, the **syscall** instruction is executed.