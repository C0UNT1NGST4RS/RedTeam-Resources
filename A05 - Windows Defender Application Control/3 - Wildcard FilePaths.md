Wildcards in FilePath rules can be quite dangerous (for obvious reasons).  For example, this rule would allow anything inside `C:\Temp\` to execute (assuming C was the OS drive):

```xml
<Allow ID="ID_ALLOW_A_2_2" FriendlyName="Temp FileRule" MinimumFileVersion="0.0.0.0" FilePath="%OSDRIVE%\Temp\*" />
```

However, by default, WDAC has a policy setting enabled called **Runtime FilePath Rule Protection**.  This is an important rule, as it only permits FilePath rules for paths that are only writable by administrators.  So in the example above, if `C:\Temp\` was writable by standard users, WDAC would not allow this rule to be applied and nothing would execute from this directory at all.

If the directory was only writable by administrators, then they could bypass WDAC by dropping and executing something malicious.  Standard users would still be able to execute from the directory, but not write to it.  They could still bypass WDAC if combined with another abuse primitive, such as an arbitrary privileged write vulnerability.

This setting can be disabled in the WDAC policy.

```xml
<Rule>
  <Option>Disabled:Runtime FilePath Rule Protection</Option>
</Rule>
```

This would allow the FilePath rule to be applied even if the directory was writeable by standard users.