Let's take the following application.  All it does is call the **MessageBoxW** API, but it serves a good visual example.

```c++
#include <iostream>
#include <Windows.h>

int main()
{
  return MessageBoxW(NULL, L"This is a test", L"TEST", 0);
}
```
  

The Import Address Table (IAT) is a relatively simply lookup table within a PE structure - it contains a list of API functions imported by the PE and their associated addresses in memory.  When a PE calls an API, it does not dynamically resolve its address at runtime (e.g. with **GetProcAddress**) because that would be costly.  Instead, it grabs it from the IAT.

If we open the MessageBox app in a tool like [CFF Explorer](https://ntcore.com/?page_id=388), we can view its IAT.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/iat/iat.png)

  

You can also view the IAT of a process whilst it's running in [WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools).

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/iat/messagebox-windbg-iat.png)

  

When executed, the message box appears with the strings as we intended.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/messagebox.png)

  

When the EDR DLL is injected, it will walk the IAT and locate any APIs it wants to hook.  It will then simply replace the addresses of those API with ones that point to detour functions in its own memory space.  This can also be seen in WinDbg.  Here we can see the address is now pointing to **IAT!MessageBoxWDetour** instead of the original in user32.dll.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/iat/messagebox-windbg-iat-hooked.png)

  

When executed with the IAT hook in place, the detour method changes the strings passed to the API call.  The detour method used for this example is very simple:

```c++
EXTERN_C
int
WINAPI
HookMessageBoxW(
  _In_opt_ HWND hWnd,
  _In_opt_ LPCWSTR lpText,
  _In_opt_ LPCWSTR lpCaption,
  _In_ UINT uType)
{
  return OrigMessageBoxW(hWnd, L"Hooked baby", L"HOOKED", uType);
}
```


![](https://rto2-assets.s3.eu-west-2.amazonaws.com/edr/messagebox-hooked.png)