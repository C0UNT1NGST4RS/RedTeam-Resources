A brute-force is where we try all combinations from a given keyspace - for lowercase alphanumeric (of 3 characters), that would mean trying `aaa`, `aab`, `aac`… all the way to `zzz`. This is incredibly time consuming and not all that efficient.

A mask attack is an evolution over the brute-force and allows us to be more selective over the keyspace in certain positions.

A common pattern would be a password that starts with an uppercase, followed by lowercase and ends with a number (e.g. `Password1`). If we used a brute-force to crack this combination we'd need to use a charset which includes uppers, lowers and numbers (62 chars in total). A password length of 9 is 62^9 (13,537,086,546,263,552) combinations.

My personal desktop computer can crack NTLM at a rate of ~30GH/s (30,000,000,000/s), which would take just over **5 days** to complete the keyspace (although it would find the password much before reaching the end).

Hashcat agrees with the calculation:

```shell
Time.Started.....: Mon Sep 14 17:35:17 2020 (5 mins, 11 secs)
Time.Estimated...: Sat Sep 19 23:47:27 2020 (5 days, 6 hours)
```

  

In contrast, a Mask would allow us to attack this password pattern in a much more efficient way. For instance, instead of using the full keyspace on the first character, we can limit ourselves to uppercase only and likewise with the other positions.

This limits the combinations to 26*26*26*26*26*26*26*26*10 (2,088,270,645,760) which is several thousands times smaller. At the same cracking rate, this will complete in around **1 minute** (and in reality, the password will be found near instantly or in several seconds).

That's quite a difference!

```shell
hashcat.exe -a 3 -m 1000 C:\Temp\ntlm.txt ?u?l?l?l?l?l?l?l?d

64f12cddaa88057e06a81b54e73b949b:Password1
```

Where:

-   `-a 3` specifies the mask attack.
-   `?u?l?l?l?l?l?l?l?d` is the mask.

`hashcat --help` will show the charsets and are as follows:

```shell
? | Charset
===+=========
l | abcdefghijklmnopqrstuvwxyz
u | ABCDEFGHIJKLMNOPQRSTUVWXYZ
d | 0123456789
h | 0123456789abcdef
H | 0123456789ABCDEF
s | !"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
a | ?l?u?d?s
b | 0x00 - 0xff
```

  

You can combine these charsets within your mask for even more flexibility. It's also common for password to end with a special (such as `!`) rather than a number, but we can specify both in a mask.

```shell
hashcat.exe -a 3 -m 1000 ntlm.txt -1 ?d?s ?u?l?l?l?l?l?l?l?1

fbdcd5041c96ddbd82224270b57f11fc:Password!
```

Where:

-   `-1 ?d?s` defines a custom charset (digits and specials).
-   `?u?l?l?l?l?l?l?l?1` is the mask, where `?1` is the custom charset.