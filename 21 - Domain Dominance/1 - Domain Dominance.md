The term "domain dominance" is used to describe a state where an attacker has reached a high level of privilege in a domain (such as Domain or Enterprise Admins) and collected credential material or placed backdoors that 1) allows them to maintain that level of access almost indefinitely; and 2) makes it practically impossible that the domain or forest can ever be considered "clean" beyond reasonable doubt.

If you've ever seen this joke before, well… it's true.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-dominance/keep_calm.png)

  

However, these techniques can put the "golden rule" at risk for red teams, because you cannot be reasonably assured that you solely control any said backdoors. Most involve granting special DACLs on objects that allow specific principals to perform actions (such as dcsync), that they wouldn't normally be able to. Even if you think you control the principal in question, you probably don't.

As such, you should only carry these out in controlled conditions with the expressed permission of the client.