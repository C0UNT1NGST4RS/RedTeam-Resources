Active Directory Certificate Services (AD CS) is a server role that allows you to build a public key infrastructure (PKI).  This can provide public key cryptography, digital certificates, and digital signature capabilities.  Some practical applications include Secure/Multipurpose Internet Mail Extensions (S/MIME), secure wireless networks, virtual private network (VPN), Internet Protocol security (IPsec), Encrypting File System (EFS), smart card logon, and Secure Socket Layer/Transport Layer Security (SSL/TLS).

Correct implementation can improve the security of an organisation:

-   Confidentiality through encryption.
-   Integrity through digital signatures.
-   Authentication by associating certificate keys with computer, user, or device accounts on the network.

  

However, like any technology, misconfigurations can introduce security risks that actors can exploit - in this case, for privilege escalation (even domain user to domain admin) and persistence.  The content found in this module is derived from the [143-page whitepaper](https://www.specterops.io/assets/resources/Certified_Pre-Owned.pdf) published by [Will Schroeder](https://twitter.com/harmj0y) & [Lee Christensen](https://twitter.com/tifkin_).  For obvious reasons, the entire content will not be duplicated here - if you want all the low-level details, do go and read it.