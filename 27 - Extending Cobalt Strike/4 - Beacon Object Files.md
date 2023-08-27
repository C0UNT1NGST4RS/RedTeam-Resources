Beacon Object Files (BOFs) are a post-ex capability that allows for code execution inside the Beacon host process.  The main advantage is to avoid the fork & run pattern that commands such as `powershell`, `powerpick` and `execute-assembly` rely on.  Since these spawn a sacrificial process and use process injection to run the post-ex action, they are heavily scrutinised by AV and EDR products.

BOFs are essentially tiny [COFF](https://en.wikipedia.org/wiki/COFF) objects (written in C or C++) on which Beacon acts as a linker and loader.  Beacon does not link BOFs to a standard C library, so many common functions are not available.  However, Beacon does expose several internal APIs that can be utilised to simplify some actions, such as argument parsing and handling output.  Note that because they run inside the Beacon process, an unstable BOF may crash your Beacon.  Run with care, or in a Beacon you don't mind losing.

To get started with writing a BOF, you'll want the Beacon header file - [beacon.h](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/beacon.h) (freely available to download), which defines the internal Beacon APIs that can be called.

>  beacon.h can be found on both the **attacker-windows** and **attacker-kali** machines, under `C:\Tools\cobaltstrike\beacon.h` and `/opt/cobaltstrike/beacon.h` respectively.    
A BOF can be built using both the Visual Studio command line and MinGW compilers.  For that reason, it doesn't matter if you dev on Windows or Linux, but it's quicker to build/test on the same machine you're running the CS GUI.

  

## Hello World

This is a basic BOF which sends the string "Hello World" as output.

```c 
#include <windows.h>
#include "beacon.h"

void go(char * args, int len)
{
    BeaconPrintf(CALLBACK_OUTPUT, "Hello World");
}
```

To compile on Windows, open the x64 Native Tools Command Prompt for VS 2019, ensure your `hello-world.c` and `beacon.h` files are in the same directory, then run: `cl.exe /c /GS- hello-world.c /Fohello-world.o`.  
If on Linux, then: `x86_64-w64-mingw32-gcc -c hello-world.c -o hello-world.o`.

To execute the BOF, run: `inline-execute \path\to\hello-world.o`.

![](https://rto-assets.s3.eu-west-2.amazonaws.com/bofs/hello-world.png)


>  This built-in `inline-execute` command expects that the entry point of the BOF is called **go**.  Otherwise, you will see an error like:  

```
[-] linker errors for UserSpecified:
Entry function 'go' is not defined.
```

  

BOFs can be integrated with aggressor by registering custom aliases and commands.

```
alias hello-world {
    local('$handle $bof $args');
    
    # read the bof file (assuming x64 only)
    $handle = openf(script_resource("hello-world.o"));
    $bof = readb($handle, -1);
    closef($handle);
    
    # print task to console
    btask($1, "Running Hello World BOF");
    
    # execute bof
    beacon_inline_execute($1, $bof, "go");
}

# register a custom command
beacon_command_register("hello-world", "Execute Hello World BOF", "Loads hello-world.o and calls the \"go\" entry point.");
```
  
The third argument of `beacon_inline_execute` is the entry point of the BOF.  If you use something other than "go", specify it here.

![](https://rto-assets.s3.eu-west-2.amazonaws.com/bofs/hello-world-cna.png)

## Handling Arguments

Naturally, there will be times where we want to pass arguments down to a BOF.  A typical console application may have an entry point which looks like: `main(int argc, char *argv[])`.  But a BOF uses `go(char * args, int len)`.  These arguments are "packed" into a special binary format using the [bof_pack](https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics_aggressor-scripts/as-resources_functions.htm#bof_pack) aggressor function, and can be "unpacked" using Beacon APIs exported via beacon.h.  Don't try and unpack these yourself if you value your sanity.

Let's work on an example where we want to provide a username to our BOF.

First, call the **BeaconDataParse** API to initialise the parser; then **BeaconDataExtract** to extract the username argument.

```
void go(char * args, int len)
{
    datap parser;
    BeaconDataParse(&parser, args, len);

    char * username;
    username = BeaconDataExtract(&parser, NULL);

    BeaconPrintf(CALLBACK_OUTPUT, "The username is: %s", username);
}
```


Arguments should be unpacked in the same order they were packed.  For instance, if we were sending both a username and password to the BOF, we would do:

```
char * username;
char * password;

username = BeaconDataExtract(&parser, NULL);
password = BeaconDataExtract(&parser, NULL);
```
  

You may also extract integers with **BeaconDataInt**.  For example if you wanted to sent a PID as an argument.

```
int pid;
pid = BeaconDataInt(&parser);
```

Arguments passed on the CS GUI command line are separated by whitespace.  The first argument, $1, is always the current Beacon, then $2, $3, $n, is your input.  We can pack the arguments we want to send in our aggressor script like so:

```
$args = bof_pack($1, "z", $2);
```

-   $2 will simply hold the username.
-   "z" tells Cobalt Strike what format of data this is, where "z" represents a zero-terminated and encoded string.

  

The Cobalt Strike documentation provides the following table of valid data formats and how to unpack them:

|Format|Description|Unpack Function|
|:---:|:---:|:---:|
|b|Binary data|BeaconDataExtract|
|i|4-byte integer (int)|BeaconDataInt|
|z|zero-terminated+encoded string|BeaconDataExtract|
|Z|zero-terminated wide string|(wchar_t *)BeaconDataExtract|

Multiple arguments should be packed at the same time:

```
// pack 2 strings
$args = bof_pack($1, "zz", "str1", "str2");

// pack a string and an int
$args = bof_pack($1, "zi", "str1", 123);
```

`beacon_inline_execute` accepts a forth parameter, of packed args.

```
beacon_inline_execute($1, $bof, "go", $args);
```

```
beacon> arg-demo imaginaryUser
[+] host called home, sent: 207 bytes
[+] received output:
The username is: imaginaryUser
```

## Calling Win32 APIs

APIs such as LoadLibrary and GetProcAddress are available from a BOF, which can be used to resolve and call other APIs at runtime.  However, BOFs provide a convention called Dynamic Function Resolution (DFR), which allows Beacon to perform the necessary resolution for you.

The definition for a DFR is best shown with an example.  Here is what it would look like to call MessageBoxW from user32.dll.

DECLSPEC_IMPORT INT WINAPI USER32$MessageBoxW(HWND, LPCWSTR, LPCWSTR, UINT);

  

Most of this information comes from the official API [documentation](https://docs.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxw), whilst DECLSPEC_IMPORT and WINAPI provide important hints for the compiler.  Putting it together:

```c
#include <windows.h>
#include "beacon.h"

void go(char * args, int len)
{
    DECLSPEC_IMPORT INT WINAPI USER32$MessageBoxW(HWND, LPCWSTR, LPCWSTR, UINT);

    datap parser;
    BeaconDataParse(&parser, args, len);

    wchar_t * message;
    message = (wchar_t *)BeaconDataExtract(&parser, NULL);

    USER32$MessageBoxW(NULL, message, L"Message Box", 0);
}
```

  

```c
alias message-box {
    local('$handle $bof $args');
    
    # read the bof file
    $handle = openf(script_resource("msg-box.o"));
    $bof = readb($handle, -1);
    closef($handle);

    # pack args
    $args = bof_pack($1, "Z", $2);
    
    # print task to console
    btask($1, "Running MessageBoxW BOF");
    
    # execute bof
    beacon_inline_execute($1, $bof, "go", $args);
}
# register a custom command
beacon_command_register("message-box", "Pop a message box", "Calls the MessageBoxW Win32 API.");
```  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/bofs/message-box.png)

There are some excellent BOFs out there which I encourage you to experiment with.  To name a few:

-   [CS-Situational-Awareness-BOF](https://github.com/trustedsec/CS-Situational-Awareness-BOF) by [TrustedSec](https://twitter.com/TrustedSec).
-   [BOF.NET](https://github.com/CCob/BOF.NET) by [@_EthicalChaos_](https://twitter.com/_EthicalChaos_).
-   [NanoDump](https://github.com/helpsystems/nanodump) by [HelpSystems](https://twitter.com/HelpSystemsMN).
-   [InlineWhispers](https://github.com/outflanknl/InlineWhispers) by [Outflank](https://twitter.com/outflanknl).