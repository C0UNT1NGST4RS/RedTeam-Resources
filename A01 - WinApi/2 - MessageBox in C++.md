Open Visual Studio and create a new C++ Console App.  The default project template outputs "Hello World" to the console.  We can just delete this line.

Add another "include" statement at the top for `<Windows.h>`.  This is a [header file](https://docs.microsoft.com/en-us/cpp/cpp/header-files-cpp) which contains declarations for all of the functions in the Windows API, all the common macros used by Windows programmers, and all the data types used by the various functions.  After adding it, you can start typing "MessageBox" and Visual Studio's Intellisense will present some options for you.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/macros.png)

  

There are so many options that it can be confusing - who knew there were so many ways to pop a message box...  The suggestions with the purple cube icons are **Functions**, and the ones with the blue/white play-button style icons are **Macros**.  These macros are like shortcuts to existing functions - in this case, the MessageBox macro points to MessageBoxW.  You can also see that MessageBoxA exists, so what's the difference?

The "A" functions use ANSI strings and "W" functions use Unicode.  Unicode is the preferred character encoding on Windows, which is why the MessageBox macro points to MessageBoxW by default.  If you look at the function definitions for MessageBoxA and MessageBoxW, you'll see that MessageBoxA takes in LPCSTR and MessageBoxW takes LPCWSTR.  If the API also returns a string (MessageBox returns an int), then the return type would also be ANSI or Unicode depending on which version of the API is called.

So we're perfectly happy to use the default macro, we can just call it like so:

```c++
#include <iostream>
#include <Windows.h>

int main()
{
    MessageBox(NULL, L"My first API call", L"Hello World", 0);
    return 0;
}
```

  

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/win32/messagebox.png)