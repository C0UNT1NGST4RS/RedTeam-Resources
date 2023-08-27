To find AD CS Certificate Authorities (CA's) in a domain or forest, run [Certify](https://github.com/GhostPack/Certify) with the cas parameter.

```
beacon> execute-assembly C:\Tools\Certify\Certify\bin\Release\Certify.exe cas
```

  

This will output lots of useful information, including the Root CAs:

```
[*] Root CAs

    Cert SubjectName              : CN=ca-1, DC=cyberbotic, DC=io
    Cert Thumbprint               : 7F8A1EFB7A50E2D1DE098085301926AA13AE0A71
    Cert Serial                   : 31AC83C6678F28994CFB58207C9FB668
    Cert Start Date               : 2/25/2022 11:29:14 AM
    Cert End Date                 : 2/25/2047 11:39:08 AM
    Cert Chain                    : CN=ca-1,DC=cyberbotic,DC=io
```

  

And enrollment CAs:

```
[*] Enterprise/Enrollment CAs:

    Enterprise CA Name            : ca-1
    DNS Hostname                  : dc-1.cyberbotic.io
    FullName                      : dc-1.cyberbotic.io\ca-1
    Flags                         : SUPPORTS_NT_AUTHENTICATION, CA_SERVERTYPE_ADVANCED
    Cert SubjectName              : CN=ca-1, DC=cyberbotic, DC=io
    Cert Thumbprint               : 7F8A1EFB7A50E2D1DE098085301926AA13AE0A71
    Cert Serial                   : 31AC83C6678F28994CFB58207C9FB668
    Cert Start Date               : 2/25/2022 11:29:14 AM
    Cert End Date                 : 2/25/2047 11:39:08 AM
    Cert Chain                    : CN=ca-1,DC=cyberbotic,DC=io

    Enterprise CA Name            : ca-2
    DNS Hostname                  : dc-2.dev.cyberbotic.io
    FullName                      : dc-2.dev.cyberbotic.io\ca-2
    Flags                         : SUPPORTS_NT_AUTHENTICATION, CA_SERVERTYPE_ADVANCED
    Cert SubjectName              : CN=ca-2, DC=dev, DC=cyberbotic, DC=io
    Cert Thumbprint               : 2D0349C77D35808E35A7C6815CF37B51D9A5D431
    Cert Serial                   : 64000000067ED180604220703C000000000006
    Cert Start Date               : 3/1/2022 10:45:07 AM
    Cert End Date                 : 3/1/2024 10:55:07 AM
    Cert Chain                    : CN=ca-1,DC=cyberbotic,DC=io -> CN=ca-2,DC=dev,DC=cyberbotic,DC=io
```

The Cert Chain is useful to note, as this shows us that CA-2 in the DEV domain is a subordinate of CA-1 in the CYBER domain.  The output will also list the certificate templates that are available at each CA, as well as some information about which principals are allowed to manage them.