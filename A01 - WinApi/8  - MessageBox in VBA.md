P/Invoke function signatures are declared a little differently in VBA - instead of a DllImport attribute, we use a `Declare` directive.  The rest is similar, in that we declare the parameters along with their VBA data types and the return type comes at the end.

Declare PtrSafe Function MessageBoxW Lib "user32.dll" (ByVal hWnd As LongPtr, ByVal lpText As String, ByVal lpCaption As String, ByVal uType As Integer) As Integer

  

Calling this function can be done in a VBA method.

Declare PtrSafe Function MessageBoxW Lib "user32.dll" (ByVal hWnd As LongPtr, ByVal lpText As String, ByVal lpCaption As String, ByVal uType As Integer) As Integer

```vb
Sub Test()
    Dim result As Integer
    result = MessageBoxW(0, StrConv("P/Invoke from MS Word!", vbUnicode), StrConv("Hello World", vbUnicode), 0)
End Sub
```

  

Because we're calling the unicode version, we need `StrConv` to convert the strings to the appropriate format.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/mesagebox-pinvoke-vba.png)