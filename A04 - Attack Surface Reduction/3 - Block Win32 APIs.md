_"This rule prevents VBA macros from calling Win32 APIs."_

Many popular tools, such as [VBA-RunPE](https://github.com/itm4n/VBA-RunPE), use P/Invoke within VBA to carry out actions such as process injection.  The way this rule is implemented seems rather odd.  If you create a word document and attempt to pop a message box, it will work.

```vb
Private Declare PtrSafe Function MessageBoxA Lib "user32.dll" (ByVal hWnd As Long, ByVal lpText As String, ByVal lpCaption As String, ByVal uType As Long) As Long

Sub Exec()
    Dim Result As Long
    Result = MessageBoxA(0, "This is a test", "Hello World", 0)
End Sub
```
  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/block-pinvoke-msgbox.png)

  

However, as soon as you try to save it to disk, Defender will block it.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/pinvoke-blocked-save.png)

  

I believe the rationale is that whenever you're sent a file, or you download one via a browser, Defender will delete it before it can be saved.  Even if you "open" a document from a browser, rather than "save" it, a temporary file is dropped to disk.  After some testing, I found that this rule only seems to be enforced when you P/Invoke directly from VBA - and not when it's done via [GadgetToJScript](https://github.com/med0x2e/GadgetToJScript).

Create a new .cs file and enter the code you want to execute.  G2JS requires the code to execute be in the constructor.

```csharp
using System;
using System.Runtime.InteropServices;

public class Program
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    static extern int MessageBoxW(IntPtr hWnd, string lpText, string lpCaption, uint uType);

    public Program()
    {
        MessageBoxW(IntPtr.Zero, "Hello from GadgetToJScript", "Hello World", 0);
    }
}
```

  

When we run G2JS, we provide the source code and specify the output type.

```powershell
C:\Tools\GadgetToJScript>GadgetToJScript\bin\Debug\GadgetToJScript.exe -c TestAssembly\Program.cs -o C:\Users\Administrator\Desktop\vba -w vba -b
[+]: Generating the vba payload
[+]: First stage gadget generation done.
[+]: Compiling your .NET code located at:TestAssembly\Program.cs
[+]: Second stage gadget generation done.
[*]: Payload generation completed, check: C:\Users\Administrator\Desktop\vba.vba
```

  

Copy the content of the VBA file, paste it into Word on WKSTN-2 and execute.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/asr/gadgettojscript.png)

  

This document will save to disk - although Defender may complain about the content, it's thanks to a static G2JS signature, rather than anything to do with P/Invoke or APIs.