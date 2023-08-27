AppLocker is Microsoft's application whitelisting technology that can restrict the executables, libraries and scripts that are permitted to run on a system. AppLocker rules are split into 5 categories - Executable, Windows Installer, Script, Packaged App and DLLs, and each category can have its own enforcement (enforced, audit only, none).

If an AppLocker category is enforced, then by default everything within that category is blocked. Rules can then be added to allow principals to execute files within that category based on a set of criteria. The rules themselves can be defined based on file attributes such as path, publisher or hash. AppLocker has a set of default allow rules such as, "allow everyone to execute anything within `C:\Windows\*`" - the theory being that everything in `C:\Windows` is trusted and safe to execute.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/applocker/default-exe-rules.png)

  

Specific deny rules can be used to override allow rules, which are commonly used to block ["LOLBAS's"](https://lolbas-project.github.io/).

Take [wmic](https://lolbas-project.github.io/lolbas/Binaries/Wmic/) as an example - even though it's a "trusted" native Windows utility, it can be used to execute "untrusted" code that would bypass AppLocker. So a deny rule for `wmic.exe` would supersede the allow rule mentioned above.

Trying to execute anything that is blocked by AppLocker looks like this:

```
C:\>test.exe
This program is blocked by group policy. For more information, contact your system administrator.
```