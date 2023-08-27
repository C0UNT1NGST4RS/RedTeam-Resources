There are a number of external utilities that are separate from the main hashcat application. Here we'll review one called [kwprocessor](https://github.com/hashcat/kwprocessor).  This is a utility for generating key-walk passwords, which are based on adjacent keys such as `qwerty`, `1q2w3e4r`, `6yHnMjU7` and so on. To humans, these can look rather random and secure (uppers, lowers, numbers & specials), but in reality they're easy to generate programmatically.

kwprocessor has three main components:

1.  Base characters - the alphabet of the target language.
2.  Keymaps - the keyboard layout.
3.  Routes - the directions to walk in.

There are several examples provided in the **basechars**, **keymaps** and **routes** directory in the kwprocessor download.

kwp64.exe basechars\custom.base keymaps\uk.keymap routes\2-to-10-max-3-direction-changes.route -o keywalk.txt

```powershell
PS C:\> Select-String -Pattern "^qwerty$" -Path keywalk.txt -CaseSensitive

D:\Tools\keywalk.txt:759:qwerty
D:\Tools\keywalk.txt:926:qwerty
D:\Tools\keywalk.txt:931:qwerty
D:\Tools\keywalk.txt:943:qwerty
D:\Tools\keywalk.txt:946:qwerty
```

  

Some candidates will get generated multiple times, so you'll want to de-dup the list before using it for maximum efficiency.  This wordlist can then be used like any other dictionary in hashcat.

> Use `kwp64.exe --help` to see customisable options such as toggling the shift key.