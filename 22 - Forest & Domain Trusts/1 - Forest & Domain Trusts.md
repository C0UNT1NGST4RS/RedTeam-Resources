At a basic level, a trust relationship enables users in one domain to authenticate and access resources in another domain. This works by allowing authentication traffic to flow between them using referrals. When a user requests access to a resource outside of their current domain, their KDC will return a referral ticket pointing to the KDC of the target domain. The user's TGT is encrypted using an inter-realm trust key (rather than the local krbtgt), which is often called an inter-realm TGT. The foreign domain decrypts this ticket, recovers the user's TGT and decides whether they should be granted access to the resource or not.

Trusts can be **one-way** or **two-way**; and **transitive** or **non-transitive**.

A one-way trust allows principals in the **trusted** domain to access resources in the **trusting** domain, but not vice versa. A two-way trust is actually just two one-way trusts that go in the opposite directions, and allows users in each domain to access resources in the other. Trust directions are confusing as the direction of the trust is the opposite to the direction of access.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/domain-trusts/trust_direction.png)

  

If Domain A trusts Domain B, Domain A is the trusting domain and Domain B is the trusted domain. But this allows users in Domain B to access Domain A, not A to B. To further complicate things, one-way trusts can be labelled as **Inbound** or **Outbound** depending on your perspective. In Domain A, this would be an Outbound trust; and in Domain B, this would be an Inbound trust.

Transitivity defines whether or not a trust can be chained. For instance - if Domain A trusts Domain B, and Domain B trusts Domain C; then A also trust C. If these domains are owned by different entities, then there are obvious implications in terms of the trust model…