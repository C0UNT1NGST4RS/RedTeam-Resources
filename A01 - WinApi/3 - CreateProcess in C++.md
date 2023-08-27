The CreateProcess API requires us to utilise some additional data structures, namely `STARTUPINFO` and `PROCESS_INFORMATION`.

```c++
#include <iostream>
#include <Windows.h>

int main()
{
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
}
```

  

Placing the cursor on these variable types (e.g. STARTUPINFO) and pressing **F12** will take you to the type definition, so that you can see what properties it has.  The unicode STARTUPINFOW looks like this:

```c++
typedef struct _STARTUPINFOW {
    DWORD   cb;
    LPWSTR  lpReserved;
    LPWSTR  lpDesktop;
    LPWSTR  lpTitle;
    DWORD   dwX;
    DWORD   dwY;
    DWORD   dwXSize;
    DWORD   dwYSize;
    DWORD   dwXCountChars;
    DWORD   dwYCountChars;
    DWORD   dwFillAttribute;
    DWORD   dwFlags;
    WORD    wShowWindow;
    WORD    cbReserved2;
    LPBYTE  lpReserved2;
    HANDLE  hStdInput;
    HANDLE  hStdOutput;
    HANDLE  hStdError;
} STARTUPINFOW, *LPSTARTUPINFOW;
```

  

And PROCESS_INFORMATION like this:

```c++
typedef struct _PROCESS_INFORMATION {
    HANDLE hProcess;
    HANDLE hThread;
    DWORD dwProcessId;
    DWORD dwThreadId;
} PROCESS_INFORMATION, *PPROCESS_INFORMATION, *LPPROCESS_INFORMATION;
```

  

We should always read the documentation for the APIs to get information on how they should be used.  For instance, the `cb` member on STARTUPINFO should contain the size of the structure (taken from the [CreateProcessW](https://docs.microsoft.com/en-us/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessw) doc).

```c++
int main()
{
    STARTUPINFO si;
    si.cb = sizeof(si);

    PROCESS_INFORMATION pi;
}
```

  

We should also zero out the memory regions for these variables to ensure there is no data in them prior to their use.

```c++
int main()
{
    STARTUPINFO si;
    si.cb = sizeof(si);
    ZeroMemory(&si, sizeof(si));

    PROCESS_INFORMATION pi;
    ZeroMemory(&pi, sizeof(pi));
}
```

  

Now we're ready to call CreateProcess.  If successful, the `pi` structure will be populated with the information from the new process.

```c++
#include <iostream>
#include <Windows.h>

int main()
{
    STARTUPINFO si;
    si.cb = sizeof(si);
    ZeroMemory(&si, sizeof(si));

    PROCESS_INFORMATION pi;
    ZeroMemory(&pi, sizeof(pi));

    BOOL success = CreateProcess(
        L"C:\\Windows\\System32\\notepad.exe",
        NULL,
        0,
        0,
        FALSE,
        0,
        NULL,
        L"C:\\Windows\\System32",
        &si,
        &pi);

    if (success)
    {
        printf("Process created with PID: %d.\n", pi.dwProcessId);
        return 0;
    }
    else
    {
        printf("Failed to create process. Error code: %d.\n", GetLastError());
        return 1;
    }
}
```

```shell
C:\Users\Administrator\source\repos\HelloWorld\x64\Debug>HelloWorld.exe
Process created with PID: 6820.
```