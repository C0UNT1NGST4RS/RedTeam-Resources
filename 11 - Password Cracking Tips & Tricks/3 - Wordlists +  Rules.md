Rules are a means of extending or manipulating the "base" words in a wordlist in ways that are common habits for users. Such manipulation can include toggling character cases (e.g. `a` to `A`), character replacement (e.g. `a` to `@`) and prepending/appending characters (e.g. `password` to `password!`).  This allows our wordlists to be overall smaller in size (because we don't have to store every permutation), but with the drawback of a slightly slower cracking time.

```shell
hashcat.exe -a 0 -m 1000 ntlm.txt rockyou.txt -r rules\add-year.rule

acbfc03df96e93cf7294a01a6abbda33:Summer2020
```

Where:

-   `-r rules\add-year.rule` is our custom rule file

The rockyou list does not contain Summer2020, but it does contain the base word, Summer.

```powershell
PS C:\> Select-String -Pattern "^Summer2020$" -Path rockyou.txt -CaseSensitive
PS C:\> Select-String -Pattern "^Summer$" -Path rockyou.txt -CaseSensitive

rockyou.txt:16573:Summer
```

  

The [hashcat wiki](https://hashcat.net/wiki/doku.php?id=rule_based_attack) contains all the information we need to write a custom rule that will append the year 2020 to each word in rockyou.  We can see that to append a character, we use `$X` - therefore to append "2020", we just need `$2$0$2$0`.

```powershell
PS C:\> cat hashcat\rules\add-year.rule
$2$0$2$0
```

 > Hashcat also ships with lots of rule files in the rules directory that you can use.