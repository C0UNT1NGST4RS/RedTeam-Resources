AD CS logging is not enabled by default, so it's unsurprisingly common for defenders to be blind to this activity in their domain.  Logging must be specifically enabled, for which for the following options are available:

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/adcs/logging-options.png)

  

**Audit Certification Services** must also be enabled via GPO to **Success** and/or **Failure** depending on the tolerance of the organisation.

  

When a certificate request is made, the CA generates a **4886** event, "Certificate Services received a certificate request".  You can find these in Kibana with:

```
event.code: 4886
```

```
Request ID:	7
Requester:	CYBER\iyates
Attributes:	

ccm:wkstn-3.cyberbotic.io
```

  

If the request was successful, and a certification issued.  The CA generates a **4887** event.

```
Certificate Services approved a certificate request and issued a certificate.
	
Request ID:	7
Requester:	CYBER\iyates
Attributes:	

ccm:wkstn-3.cyberbotic.io
Disposition:	3
SKI:		5b c2 d9 2e 14 6c cd bc c4 b8 d6 31 18 35 07 85 b6 83 61 be
Subject:	CN=Isabel Yates, CN=Users, DC=cyberbotic, DC=io
```

  

This request was generated following the steps in "Misconfigured Certificate Templates".  You'll notice that there's very little useful information here - the template name is not logged and neither is the subject alternate name.  There's practically no indication that this is a malicious request.  A defender would have to go to the CA itself and lookup the certificate by way of the **Request ID**.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/adcs/san-detection.png)

  

Only then do we see that **iyates** obtained a certificate for **nglover**.

  

When a TGT is requested, the DC generates a **4768**.  If a certificate was used, this is shown in the log.

```
Certificate Information:
	Certificate Issuer Name:	ca-1
	Certificate Serial Number:	‎64000000077D7D5EBB98E11D17000000000007
	Certificate Thumbprint:		‎3EB5122F3B926D7591EAB07D0C9EE1BFA6977C5F
```

Again, this is not easy to correlate, as we have to go to the CA and find the actual certificate that was used by its serial number or thumbprint.  Overall, ADCS logging is not terribly easy to deal with.