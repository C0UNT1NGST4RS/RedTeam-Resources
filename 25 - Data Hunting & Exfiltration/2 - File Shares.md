`Find-DomainShare` will find SMB shares in a domain and `-CheckShareAccess` will only display those that the executing principal has access to.

```
beacon> powershell Find-DomainShare -ComputerDomain cyberbotic.io -CheckShareAccess

Name     Type Remark              ComputerName      
----     ---- ------              ------------      
data$       0                     dc-1.cyberbotic.io

beacon> ls \\dc-1.cyberbotic.io\data$

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 17kb     fil     03/25/2021 12:54:11   63ec08038c04ac60dab340ee9569e690dataMar-25-2021.xlsx
 214kb    fil     03/25/2021 13:02:53   Apple iPhone 8.ai
 1mb      fil     03/25/2021 13:02:55   Boeing 787-8 DreamLiner Air Canada.ai
 540kb    fil     03/25/2021 13:02:54   Caterpillar 345D L.ai
 829kb    fil     03/25/2021 13:02:55   Ducati 1098R (2011).ai
 895kb    fil     03/25/2021 13:02:55   Volkswagen Caddy Maxi (2020).ai
```

