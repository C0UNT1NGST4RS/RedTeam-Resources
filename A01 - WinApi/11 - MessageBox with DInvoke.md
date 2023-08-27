D/Invoke comes as a DLL that you use as a reference in your C# project.  The source code is located in `C:\Tools\DInvoke` and a pre-compiled version at `C:\Tools\DInvoke\DInvoke\DInvoke\bin\Debug\DInvoke.dll` (of course you can modify the source and compile your own version if required).

Instead of using `DllImport`, D/Invoke relies on delegates decorated with the `UnmanagedFunctionPointer` attribute.  For MessageBoxW, that would look like this:

```csharp
[UnmanagedFunctionPointer(CallingConvention.StdCall, CharSet = CharSet.Unicode)]
delegate int MessageBoxW(IntPtr hWnd, string lpText, string pCaption, uint uType);
```

  

The simplest way to execute this is with `DInvoke.DynamicInvoke.Generic.DynamicAPIInvoke`.  We must first define our input parameters within an `object[]` and then pass it in as a reference.

```csharp
using System;
using System.Runtime.InteropServices;

using DInvoke.DynamicInvoke;

namespace ConsoleApp1
{
    internal class Program
    {
        [UnmanagedFunctionPointer(CallingConvention.StdCall, CharSet = CharSet.Unicode)]
        delegate int MessageBoxW(IntPtr hWnd, string lpText, string pCaption, uint uType);

        static void Main(string[] args)
        {
            var parameters = new object[] { IntPtr.Zero, "My first D/Invoke!", "Hello World", (uint)0 };
            Generic.DynamicAPIInvoke("user32.dll", "MessageBoxW", typeof(MessageBoxW), ref parameters);
        }
    }
}
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/mesagebox-dinvoke.png)

  

Another method is to use `GetLibraryAddress` and `GetDelegateForFunctionPointer`.  This is typically more convenient if you need to call the same API more than once.

```csharp
static void Main(string[] args)
{
    var address = Generic.GetLibraryAddress("user32.dll", "MessageBoxW");
    var messageBoxW = (MessageBoxW) Marshal.GetDelegateForFunctionPointer(address, typeof(MessageBoxW));

    messageBoxW(IntPtr.Zero, "Box 1", "Box 1", 0);
    messageBoxW(IntPtr.Zero, "Box 2", "Box 2", 0);
}
```

  

One thing D/Invoke is good at is providing multiple ways to achieve the same goal, which provides lots of flexibility.  And if you open this assembly in pestudio, it won't see that we're using the MessageBoxW API.

Furthermore, `GetLibraryAddress` has an overload that will take an ordinal.

```csharp
var address = Generic.GetLibraryAddress("user32.dll", 2155);
```