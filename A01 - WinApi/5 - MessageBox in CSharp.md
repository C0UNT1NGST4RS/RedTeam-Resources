Create a new C# Console App (.NET Framework) project.

The first step is to use the `DllImport` attribute to tell the runtime it needs to load the specified unmanaged DLL.

```csharp
using System;
using System.Runtime.InteropServices;

namespace PInvoke
{
    internal class Program
    {
        [DllImport("user32.dll", CharSet = CharSet.Unicode)]
        static extern int MessageBoxW(IntPtr hWnd, string lpText, string lpCaption, uint uType);

        static void Main(string[] args)
        {
        }
    }
}
```

  

Within this attribute, we also define the character set so that the .NET runtime can correctly marshal managed strings to the correct unmanaged types.  The name of the method should match the unmanaged API that we wish to call, and the return type and input parameters are defined as well.  We can get the function signature directly from the [MessageBoxW documentation](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxw), or from a resource such as [pinvoke.net](http://pinvoke.net/default.aspx/user32/MessageBox.html).

We do have to translate unmanaged types to their managed type counterparts.  For instance, anything that's a HANDLE in C++ can be an IntPtr in C#.

Now this method can be called from Main.

```csharp
using System;
using System.Runtime.InteropServices;

namespace PInvoke
{
    internal class Program
    {
        [DllImport("user32.dll", CharSet = CharSet.Unicode)]
        static extern int MessageBoxW(IntPtr hWnd, string lpText, string lpCaption, uint uType);

        static void Main(string[] args)
        {
            MessageBoxW(IntPtr.Zero, "My first P/Invoke", "Hello World", 0);
        }
    }
}
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/mesagebox-pinvoke.png)