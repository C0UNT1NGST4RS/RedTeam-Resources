There may be instances across the domain where some principals have ACLs on more privileged accounts, that allow them to be abused for account-takeover. A simple example of this could be a "support" group that can reset the passwords of "Domain Admins".

We can start off by targeting a single principal. This query will return any principal that has **GenericAll**, **WriteProperty** or **WriteDacl** on jadams.

```
beacon> powershell Get-DomainObjectAcl -Identity jadams | ? { $_.ActiveDirectoryRights -match "GenericAll|WriteProperty|WriteDacl" -and $_.SecurityIdentifier -match "S-1-5-21-3263068140-2042698922-2891547269-[\d]{4,10}" } | select SecurityIdentifier, ActiveDirectoryRights | fl

SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125
ActiveDirectoryRights : GenericAll

SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125
ActiveDirectoryRights : GenericAll

beacon> powershell ConvertFrom-SID S-1-5-21-3263068140-2042698922-2891547269-1125
DEV\1st Line Support
```

  

We could also cast a wider net and target entire OUs.

```
beacon> powershell Get-DomainObjectAcl -SearchBase "CN=Users,DC=dev,DC=cyberbotic,DC=io" | ? { $_.ActiveDirectoryRights -match "GenericAll|WriteProperty|WriteDacl" -and $_.SecurityIdentifier -match "S-1-5-21-3263068140-2042698922-2891547269-[\d]{4,10}" } | select ObjectDN, ActiveDirectoryRights, SecurityIdentifier | fl

ObjectDN              : CN=Joyce Adam,CN=Users,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : GenericAll
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN              : CN=1st Line Support,CN=Users,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : GenericAll
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN              : CN=Developers,CN=Users,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : GenericAll
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN              : CN=Oracle Admins,CN=Users,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : GenericAll
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125
```
  

In BloodHound, this query returns a lot of information which can look a bit confusing.

```
MATCH (g1:Group), (g2:Group), p=((g1)-[:GenericAll]->(g2)) RETURN p
```

  

We can narrow it down with:

```
MATCH (g1:Group {name:"1ST LINE SUPPORT@DEV.CYBERBOTIC.IO"}), (g2:Group), p=((g1)-[:GenericAll]->(g2)) RETURN p
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/dacl/generic-all.png)

  

This shows that 1st Line Support has **GenericAll** on multiple users and groups. So how can we abuse these?