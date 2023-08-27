Applications that are not signed are often permitted to execute based on their full path.  For instance, this demo binary in `C:\Program Files\LegitApp\`.

```powershell
PS C:\> Get-AuthenticodeSignature -FilePath 'C:\Program Files\LegitApp\*' | ft

SignerCertificate                         Status                                 Path
-----------------                         ------                                 ----
                                          NotSigned                              LegitApp.exe
                                          NotSigned                              LegitApp.dll
```

The policy rules could look like this:

```xml
<FileRules>
  <Allow ID="ID_ALLOW_A_1" FriendlyName="LegitApp.exe FileRule" MinimumFileVersion="0.0.0.0" FilePath="C:\Program Files\LegitApp\LegitApp.exe" />
  <Allow ID="ID_ALLOW_A_2" FriendlyName="LegitApp.dll FileRule" MinimumFileVersion="0.0.0.0" FilePath="C:\Program Files\LegitApp\LegitApp.dll" />
</FileRules>
```

When executed, this app just opens a message box.

If a user had permission to modify/replace either of these files, it would still be allowed to execute because the file path is still the same, and that's all that's being checked.  It would be quite unlikely for a standard user but possible for a local admin, and some applications are installed in user-writable locations, such as `AppData`.

```powershell
PS C:\> Get-Acl -Path 'C:\Program Files\LegitApp\LegitApp.dll' | select -expand Access

FileSystemRights  : Modify, Synchronize
AccessControlType : Allow
IdentityReference : NT AUTHORITY\Authenticated Users
IsInherited       : False
InheritanceFlags  : None
PropagationFlags  : None
```

  

Runtime FilePath Rule Protection does apply to DACLs on individual files, so needs to be disabled in order for standard users to exploit this.