By default, this mask attack sets a static password length - `?u?l?l?l?l?l?l?l?1` defines 9 characters, which means we can only crack a 9-character password. To crack passwords of different lengths, we have to manually adjust the mask accordingly.

Hashcat mask files make this process a lot easier for custom masks that you use often.

```powershell
PS C:\> cat example.hcmask
?d?s,?u?l?l?l?l?1
?d?s,?u?l?l?l?l?l?1
?d?s,?u?l?l?l?l?l?l?1
?d?s,?u?l?l?l?l?l?l?l?1
?d?s,?u?l?l?l?l?l?l?l?l?1
```

```shell
hashcat.exe -a 3 -m 1000 ntlm.txt example.hcmask
hashcat (v6.1.1) starting...

Status...........: Exhausted
Guess.Mask.......: ?u?l?l?l?l?1 [6]

[...snip...]

Guess.Mask.......: ?u?l?l?l?l?l?1 [7]

820be3700dfcfc49e6eb6ef88d765d01:Chimney!
```

  

Masks can even have static strings defined, such as a company name or other keyword you suspect are being used in passwords.

```shell
ZeroPointSecurity?d
ZeroPointSecurity?d?d
ZeroPointSecurity?d?d?d
ZeroPointSecurity?d?d?d?d
```

```shell
hashcat.exe -a 3 -m 1000 ntlm.txt example2.hcmask

f63ebb17e157149b6dfde5d0cc32803c:ZeroPointSecurity1234
```