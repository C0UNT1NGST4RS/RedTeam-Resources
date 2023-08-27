Before we can call the CreateProcess API, we have to define the `STARTUPINFO` and `PROCESS_INFORMATION` data structures.  We do this with the `struct` type in C#.

```csharp
using System;
using System.Runtime.InteropServices;

namespace PInvoke
{
    internal class Program
    {
        [StructLayout(LayoutKind.Sequential)]
        public struct STARTUPINFO
        {
            public int cb;
            public IntPtr lpReserved;
            public IntPtr lpDesktop;
            public IntPtr lpTitle;
            public int dwX;
            public int dwY;
            public int dwXSize;
            public int dwYSize;
            public int dwXCountChars;
            public int dwYCountChars;
            public int dwFillAttribute;
            public int dwFlags;
            public short wShowWindow;
            public short cbReserved2;
            public IntPtr lpReserved2;
            public IntPtr hStdInput;
            public IntPtr hStdOutput;
            public IntPtr hStdError;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct PROCESS_INFORMATION
        {
            public IntPtr hProcess;
            public IntPtr hThread;
            public int dwProcessId;
            public int dwThreadId;
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct SECURITY_ATTRIBUTES
        {
            public int nLength;
            public IntPtr lpSecurityDescriptor;
            public bool bInheritHandle;
        }

        [DllImport("kernel32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        static extern bool CreateProcessW(string lpApplicationName, string lpCommandLine, ref SECURITY_ATTRIBUTES lpProcessAttributes,
            ref SECURITY_ATTRIBUTES lpThreadAttributes, bool bInheritHandles, uint dwCreationFlags, IntPtr lpEnvironment,
            string lpCurrentDirectory, ref STARTUPINFO lpStartupInfo, out PROCESS_INFORMATION lpProcessInformation);

        static void Main(string[] args)
        {
        }
    }
}
```

  

You can see this is a lot of extra work when compared to C++.  Some aspects to note about this DllImport attribute:

-   Setting `SetLastError` to true tells the runtime to capture the error code.  This allows the user to retrieve it using `Marshal.GetLastWin32Error()`.
-   Reading the API documentation, you will see that some parameters are passed in as pointers (e.g. `&si` in C++).  In these cases, we use the `ref` keyword.  For data marshalled out of the API call (e.g. for PROCESS_INFORMATION), we use the `out` keyword.

We can now call the API:

```csharp
static void Main(string[] args)
{
    var si = new STARTUPINFO();
    si.cb = Marshal.SizeOf(si);

    var pa = new SECURITY_ATTRIBUTES();
    pa.nLength = Marshal.SizeOf(pa);

    var ta = new SECURITY_ATTRIBUTES();
    ta.nLength = Marshal.SizeOf(ta);

    var pi = new PROCESS_INFORMATION();

    var success = CreateProcessW(
        "C:\\Windows\\System32\\notepad.exe",
        null,
        ref pa,
        ref ta,
        false,
        0,
        IntPtr.Zero,
        "C:\\Windows\\System32",
        ref si,
        out pi);

    if (success)
        Console.WriteLine("Process created with PID: {0}.", pi.dwProcessId);
    else
        Console.WriteLine("Failed to create process. Error code: {0}.", Marshal.GetLastWin32Error());
}
```

```shell
C:\Users\Administrator\source\repos\PInvoke\bin\Debug>PInvoke.exe
Process created with PID: 8412.
```