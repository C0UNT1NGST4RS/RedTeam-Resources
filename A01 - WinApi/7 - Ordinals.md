There are multiple ways to define this DllImport attribute which can be useful for circumventing some AV signatures.  This is the way we're currently using them:

```csharp
[DllImport("user32.dll", CharSet = CharSet.Unicode)]
static extern int MessageBoxW(IntPtr hWnd, string lpText, string lpCaption, uint uType);
```

  

As previously stated, the name of the method must match the name of the API.  In this case, we have to use MessageBoxW, because that's the API we want to call.  If an AV engine was specifically looking at these, then this would be detected.  So can we call an API without using its name?

Yes - enter ordinals.

An ordinal is a number that identifies an exported function in a DLL - think of them as the Primary Key in a database table.  Each exported function has an associated ordinal which is unique in that DLL, and we can use these ordinals with DllImport.

To find the ordinal of an exported function, open the DLL with **PEview.exe,** find the **EXPORT Address Table** and scroll to the exported function that you want to call.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/peview.png)

  

The ordinal in hex for MessageBoxW is **086B**.  Convert this to decimal using a calculator in programmer mode or an online converter, and you get **2155**.  Now we can change the DllImport attribute to be something like this:

```csharp
[DllImport("user32.dll", EntryPoint = "#2155", CharSet = CharSet.Unicode)]
static extern int TotallyLegitAPI(IntPtr hWnd, string lpText, string lpCaption, uint uType);
```

  

The **EntryPoint** property indicates the name or ordinal of the exported function to call, but since we want to avoid using the name, use the ordinal.  We can then execute **TotallyLegitAPI** and pass in all the same parameters and it will work as expected.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/messageboxw-ordinal.png)

  

  > Ordinal numbers can vary across Windows versions, so ensure you're using the correct ones for your target.