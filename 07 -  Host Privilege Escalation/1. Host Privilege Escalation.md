Host privilege escalation allows us to elevate privileges from that of a standard user to Administrator. It’s not a necessary step, as we’ll see in later modules how it's possible to obtain privileged credentials and move laterally in the domain without having to "priv esc" first. 

However, elevated privileges can provide a tactical advantage by allowing you to leverage some additional capabilities. For example, dumping credentials with Mimikatz, installing sneaky persistence or manipulating host configuration such as the firewall.

In keeping with the mantra of "principle of least privilege" - privilege escalation should only be sought after if it provides a means of reaching your goal, not something you do "just because".  Exploiting a privilege escalation vulnerability provides defenders with additional data points to detect your presence.  It's a risk vs reward calculation that you must make.

Common methods for privilege escalation include Operating System or 3rd party software misconfigurations and missing patches.  [SharpUp](https://github.com/GhostPack/SharpUp) can enumerate the host for any misconfiguration-based opportunities.