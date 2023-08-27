Group Policy is the central repository in a forest or domain that controls the configuration of computers and users. Group Policy Objects (GPOs) are sets of configurations that are applied to Organisational Units (OUs). Any users or computers that are members of the OU will have those configurations applied.

By default, only Domain Admins can create GPOs and link them to OUs but it's common practice to delegate those rights to other teams, e.g. delegating workstation admins permissions to create and link GPOs to a Workstation OU.

It's relatively easy to create privilege escalation opportunities when a group of users have permissions to influence the GPOs applied to privileged users; or to a computer used by privileged users. GPOs can also be leveraged to move laterally and create persistence backdoors.

Any domain user can enumerate the permissions on GPOs and OUs - so we can find users who can:

-   Create GPOs
-   Modify existing GPOs
-   Link GPOs to OUs

We can abuse these by modifying existing GPOs or creating and linking new GPOs to gain code execution or otherwise manipulate computer configurations.

This PowerView query will show the Security Identifiers (SIDs) of principals that can create new GPOs in the domain, which can be translated via `ConvertFrom-SID`.

```
beacon> powershell Get-DomainObjectAcl -SearchBase "CN=Policies,CN=System,DC=dev,DC=cyberbotic,DC=io" -ResolveGUIDs | ? { $_.ObjectAceType -eq "Group-Policy-Container" } | select ObjectDN, ActiveDirectoryRights, SecurityIdentifier | fl

ObjectDN              : CN=Policies,CN=System,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : CreateChild
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1125

beacon> powershell ConvertFrom-SID S-1-5-21-3263068140-2042698922-2891547269-1125

DEV\1st Line Support
```

  

This query will return the principals that can write to the GP-Link attribute on OUs:

```
beacon> powershell Get-DomainOU | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.ObjectAceType -eq "GP-Link" -and $_.ActiveDirectoryRights -match "WriteProperty" } | select ObjectDN, SecurityIdentifier | fl

ObjectDN           : OU=Workstations,DC=dev,DC=cyberbotic,DC=io
SecurityIdentifier : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN           : OU=Servers,DC=dev,DC=cyberbotic,DC=io
SecurityIdentifier : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN           : OU=Tier 1,OU=Servers,DC=dev,DC=cyberbotic,DC=io
SecurityIdentifier : S-1-5-21-3263068140-2042698922-2891547269-1125

ObjectDN           : OU=Tier 2,OU=Servers,DC=dev,DC=cyberbotic,DC=io
SecurityIdentifier : S-1-5-21-3263068140-2042698922-2891547269-1125
```

  

From this output, we can see that the **1st Line Support** domain group can both create new GPOs **and** link them to several OUs. This can lead to a privilege escalation if more privileged users are authenticated to any of the machines within those OUs. Also imagine if we could link GPOs to an OU containing sensitive file or database servers - we could use those GPOs to access those machines and subsequently the data stored on them.

You can also get a list of machines within an OU.

```
beacon> powershell Get-DomainComputer | ? { $_.DistinguishedName -match "OU=Tier 1" } | select DnsHostName

dnshostname            
-----------            
srv-1.dev.cyberbotic.io
```

  

You'll often find instances where users and / or groups can modify existing GPOs.

This query will return any GPO in the domain, where a 4-digit RID has **WriteProperty**, **WriteDacl** or **WriteOwner**. Filtering on a 4-digit RID is a quick way to eliminate the default 512, 519, etc results.

```
beacon> powershell Get-DomainGPO | Get-DomainObjectAcl -ResolveGUIDs | ? { $_.ActiveDirectoryRights -match "WriteProperty|WriteDacl|WriteOwner" -and $_.SecurityIdentifier -match "S-1-5-21-3263068140-2042698922-2891547269-[\d]{4,10}" } | select ObjectDN, ActiveDirectoryRights, SecurityIdentifier | fl

ObjectDN              : CN={AD7EE1ED-CDC8-4994-AE0F-50BA8B264829},CN=Policies,CN=System,DC=dev,DC=cyberbotic,DC=io
ActiveDirectoryRights : CreateChild, DeleteChild, ReadProperty, WriteProperty, GenericExecute
SecurityIdentifier    : S-1-5-21-3263068140-2042698922-2891547269-1126

beacon> powershell ConvertFrom-SID S-1-5-21-3263068140-2042698922-2891547269-1126
DEV\Developers
```

  

To resolve the ObjectDN:

```
beacon> powershell Get-DomainGPO -Name "{AD7EE1ED-CDC8-4994-AE0F-50BA8B264829}" -Properties DisplayName


displayname       
-----------       
PowerShell Logging
```


In BloodHound:

```
MATCH (gr:Group), (gp:GPO), p=((gr)-[:GenericWrite]->(gp)) RETURN p
```

![](https://rto-assets.s3.eu-west-2.amazonaws.com/group-policy/devs-genericwrite.png)