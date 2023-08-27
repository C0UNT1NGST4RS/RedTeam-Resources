A "wordlist" or "dictionary" attack is the easiest mode of password cracking, in which we simply read in a list of password candidates and try each one line-by-line. There are many popular lists out there, including the venerable rockyou list.  The [SecLists](https://github.com/danielmiessler/SecLists/tree/master/Passwords) repo also have an expansive collection for different applications.

```shell
hashcat.exe -a 0 -m 1000 ntlm.txt rockyou.txt

58a478135a93ac3bf058a5ea0e8fdb71:Password123
```

Where:

-   `-a 0` specifies the wordlist attack mode.
-   `-m 1000` specifies that the hash is NTLM.
-   `ntlm.txt` is a text file containing the NTLM hash to crack.
-   `rockyou.txt` is the wordlist.

>  Use `hashcat.exe --help` to get a complete list of attack mode and hash types.

This cracks practically instantly because 'Password123' is present in the wordlist:

```powershell
PS C:\> Select-String -Pattern "^Password123$" -Path rockyou.txt -CaseSensitive

rockyou.txt:33523:Password123
```

  

Although fast it's not very flexible, since if the password is not in the list, we won't crack it.