If the DLL is not signed by Microsoft, we can prevent it from being loaded into processes via the **PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY**.  This is really easy to do as it can be defined in a **PPROC_THREAD_ATTRIBUTE_LIST** when spawning a process - exactly the same as we did with PPID Spoofing.

In C++, it would look something like this:

```cpp
DWORD64 policy = PROCESS_CREATION_MITIGATION_POLICY_BLOCK_NON_MICROSOFT_BINARIES_ALWAYS_ON;

if (!UpdateProcThreadAttribute(sie.lpAttributeList, 0, PROC_THREAD_ATTRIBUTE_MITIGATION_POLICY, &policy, sizeof(DWORD64), NULL, NULL))
{
	printf("UpdateProcThreadAttribute failed. Error code: %d.\n", GetLastError());
	return 0;
}
``` 

Use Process Hacker to confirm the policy has been applied to the process.  Also ensure the EDR driver is running when the process is spawned and validate that the DLL is not able to be injected.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/blockdlls/notepad-blockdlls.png)


This policy does not prevent shellcode injection, so you may write an injector that spawns a process with this policy and inject Beacon shellcode into it.

---

Since Beacon has many fork and run behaviours, it has its own `blockdlls` command, which allows it to spawn the sacrificial processes with this policy.

```shell
beacon> help blockdlls
Use: blockdlls [start|stop]

Launch child processes with a binary signature policy that blocks 
non-Microsoft DLLs from loading into the child process.

Use "blockdlls stop" to disable this behavior.

This feature requires Windows 10/Windows Server 2012 or later.
```

This will keep the EDR DLL out of fork and run post-ex commands such as `execute-assembly` and `powerpick.`