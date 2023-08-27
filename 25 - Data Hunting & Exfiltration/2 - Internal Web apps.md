Internal web apps are incredibly prevalent and are a great source of data. Think SharePoint, Confluence, ServiceNow, SIEMs and so on.

[EyeWitness](https://github.com/FortyNorthSecurity/EyeWitness) is a tool capable of identifying (and taking screenshots of) web apps from a list of targets.

```
root@kali:/opt/EyeWitness/Python# cat /root/targets.txt
10.10.17.71
10.10.17.25
10.10.17.68

root@kali:/opt/EyeWitness/Python# proxychains4 ./EyeWitness.py --web -f /root/targets.txt -d /root/dev --no-dns --no-prompt

Starting Web Requests (3 Hosts)
Attempting to screenshot http://10.10.17.71
[*] WebDriverError when connecting to http://10.10.17.71
Attempting to screenshot http://10.10.17.25
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  10.10.17.25:80  ...  OK
Attempting to screenshot http://10.10.17.68
[*] WebDriverError when connecting to http://10.10.17.68
Finished in 12.967030048370361 seconds
```

```
PS C:\Users\Administrator\Desktop> pscp -r root@kali:/root/dev .
```

![](https://rto-assets.s3.eu-west-2.amazonaws.com/data-hunting/eyewitness-report.png)
