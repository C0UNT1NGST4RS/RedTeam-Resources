There are many native Windows binaries and scripts that can be used to execute arbitrary code.  These can be used to bypass a WDAC policy which trusts signed Windows applications.  Microsoft actively maintains a [recommended blocklist](https://docs.microsoft.com/en-us/windows/security/threat-protection/windows-defender-application-control/microsoft-recommended-block-rules) to combat these, which organisations should implement.  An example of such a rule:

  <FileRules>
    <Deny ID="DENY_MSBUILD" FriendlyName="MSBuild.exe" FileName="MSBuild.Exe" MinimumFileVersion="65535.65535.65535.65535"/>
  </FileRules>

```
PS C:\Users\hdoyle> C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe /?
Program 'MSBuild.exe' failed to run: Your organization used Device Guard to block this app.
```

If you enumerate the WDAC policy and find that any of these are missing, you may be in luck.  Unfortunately, explicit details on how to use some of these applications are not public.  So unless the information happens to be available (e.g. on the [Ultimate WDAC Bypass List](https://github.com/bohops/UltimateWDACBypassList)), then you have to effectively discover the bypass technique yourself.

>  **EXERCISE**    
Find a LOLBAS that will bypass the WDAC policy on **WKSTN-3**.
