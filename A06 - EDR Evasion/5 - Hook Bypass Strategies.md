Both IAT and inline hooks can be effectively "unhooked" by patching over them to restore their original values. Since this is all happening in userland of a process we control, we're relatively free to do that.  In the case of inline hooking, we can even reload entire modules from disk (e.g. kernel32.dll, ntdll.dll etc) which would map an entirely fresh copy into memory, erasing the hooks.

One rather significant downside to this, is that EDRs do monitor the integrity of their own hooks.  So even if we did unhook them, the EDR can simply re-hook them at a later date and more significantly, raise an alert that hook tampering was detected.

In my view, a better strategy is to find different ways of executing the desired APIs, without ever touching the hooks.