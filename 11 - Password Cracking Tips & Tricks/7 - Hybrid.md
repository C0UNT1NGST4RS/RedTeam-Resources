Hashcat modes 6 and 7 are hybrid's based on wordlists, masks and the combinator.  You specify both a wordlist and mask on the command line, and the mask is appended or prepended to the words within the list. For example, your dictionary contains the word `Password`, then `-a 6 [...] list.txt ?d?d?d?d` will produce `Password0000` to `Password9999`.

```shell
hashcat.exe -a 6 -m 1000 ntlm.txt list.txt ?d?d?d?d

be4c5fb0b163f3cc57bd390cdc495bb9:Password5555
```

Where:

-   `-a 6` specifies the hybrid wordlist + mask mode.
-   `?d?d?d?d` is the mask.

The hybrid mask + wordlist mode (`-a 7`) is practically identical, where the mask comes first.

```shell
hashcat.exe -a 7 -m 1000 ntlm.txt ?d?d?d?d list.txt

28a3b8f54a6661f15007fca23beccc9c:5555Password
```