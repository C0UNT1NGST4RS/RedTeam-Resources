"Marshalling" is the process of transforming a data type when it needs to cross between managed and unmanaged code.  By default, the P/Invoke subsystem tries to automatically marshal data for you, but there may be situations where you need to marshal it manually.

In the previous lesson, we called MessageBoxW which took two `string` parameters, and we know these needed to be Unicode (LPCWSTR).  We didn't have to do anything magical, because P/Invoke did it for us.  If we needed to marshal them manually, we would add the `MarshalAs` attribute to the parameters.  That would look something like this:

```csharp
[DllImport("user32.dll")]
static extern int MessageBoxW(
    IntPtr hWnd,
    [MarshalAs(UnmanagedType.LPWStr)] string lpText,
    [MarshalAs(UnmanagedType.LPWStr)] string lpCaption,
    uint uType);
```

  

[This page](https://docs.microsoft.com/en-us/dotnet/framework/interop/marshaling-data-with-platform-invoke) contains a useful table of managed and unmanaged data type mappings.  Note that P/Invoke handles 99% of cases without issue - this information is more relevant when accessing native APIs without P/Invoke (e.g. D/Invoke).