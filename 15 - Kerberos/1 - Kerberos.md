Kerberos is a fun topic and contains some of the more well-known abuse primitives within Active Directory environments. It can also be a bit elusive as to how it works since it has so many complex intricacies, but here's a brief overview:

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/kerberos/overview.png)

  

When a user logs onto their workstation, their machine will send an **AS-REQ** message to the Key Distribution Center (KDC), aka Domain Controller, requesting a TGT using a secret key derived from the user’s password.

The KDC verifies the secret key with the password it has stored in Active Directory for that user. Once validated, it returns the TGT in an **AS-REP** message. The TGT contains the user’s identity and is encrypted with the KDC secret key (the **krbtgt** account).

When the user attempts to access a resource backed by Kerberos authentication (e.g. a file share), their machine looks up the associated Service Principal Name (SPN). It then requests (**TGS-REQ**) a Ticket Granting Service Ticket (TGS) for that service from the KDC, and presents its TGT as a means of proving they're a valid user.

The KDC returns a TGS (**TGS-REP**) for the service in question to the user, which is then presented to the actual service. The service inspects the TGS and decides whether it should grant the user access or not.