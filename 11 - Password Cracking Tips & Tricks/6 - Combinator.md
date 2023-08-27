The combinator attack combines the entries from two dictionaries into single-word candidates. Take the following lists as an example:

```powershell
PS C:\> cat list1.txt
purple

PS C:\> cat list2.txt
monkey
dishwasher
```

  

The combinator will produce "purplemonkey" and "purpledishwasher" as candidates.  You can also apply a rule to each word on the left- or right-hand side using the options `-j` and `-k`.  For instance, `-j $-` and `-k $!` would produce `purple-monkey!`.

```shell
hashcat.exe -a 1 -m 1000 ntlm.txt list1.txt list2.txt -j $- -k $!

ef81b5ffcbb0d030874022e8fb7e4229:purple-monkey!
```
  
> If running in Linux, shells (`sh`, `bash`, `zsh`, `fish`, etc) will have their own behaviour when the $ character is used on the command line.  They may need to be quoted.