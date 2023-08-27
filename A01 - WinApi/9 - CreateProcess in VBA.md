As with C#, we must define the necessary structures in VBA before we can call the API.  That can be done with the `Type` declaration.

```vb
Declare PtrSafe Function CreateProcessW Lib "kernel32.dll" (ByVal lpApplicationName As String, ByVal lpCommandLine As String, ByVal lpProcessAttributes As LongPtr, ByVal lpThreadAttributes As LongPtr, ByVal bInheritHandles As Boolean, ByVal dwCreationFlags As Long, ByVal lpEnvironment As LongPtr, ByVal lpCurrentDirectory As String, lpStartupInfo As STARTUPINFO, lpProcessInformation As PROCESS_INFORMATION) As Boolean

Type STARTUPINFO
    cb As Long
    lpReserved As String
    lpDesktop As String
    lpTitle As String
    dwX As Long
    dwY As Long
    dwXSize As Long
    dwYSize As Long
    dwXCountChars As Long
    dwYCountChars As Long
    dwFillAttribute As Long
    dwFlags As Long
    wShowWindow As Integer
    cbReserved2 As Integer
    lpReserved2 As LongPtr
    hStdInput As LongPtr
    hStdOutput As LongPtr
    hStdError As LongPtr
End Type

Type PROCESS_INFORMATION
    hProcess As LongPtr
    hThread As LongPtr
    dwProcessId As Long
    dwThreadId As Long
End Type
```
  

Then it's as simple as calling the API and plugging in the desired values.

```vb
Sub Test()
    Dim si As STARTUPINFO
    Dim pi As PROCESS_INFORMATION
    
    Dim nullStr As String
    
    Dim success As Boolean
    success = CreateProcessW(StrConv("C:\Windows\System32\notepad.exe", vbUnicode), nullStr, 0&, 0&, False, 0, 0&, nullStr, si, pi)
End Sub
```