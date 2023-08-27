As we've seen, there are numerous ways in which we can obtain credential material for a user - but this is not always in the form of a plaintext password. Instead, it's more common these days to retrieve various hashes. These could be NTLM, NetNTLM, SHA or even Kerberos tickets.

Some hashes such as NTLM can be utilised as they are (e.g. pass the hash), but others are not so useful unless we can crack them to recover an original plaintext password. Regardless of the type of hash, there are generic password cracking methodologies that we'll cover here.

Two very common applications to achieve this are [hashcat](https://hashcat.net/hashcat/) and [John the Ripper](https://www.openwall.com/john/).
> If you want to copy the example NTLM hashes to try to crack them yourself, you will need to do so on your own computer since the lab VMs are not compute optimised.