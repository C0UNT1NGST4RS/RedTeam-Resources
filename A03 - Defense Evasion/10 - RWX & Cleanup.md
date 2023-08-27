As we know, memory regions have a protection level.  When writing our process injection applications, we were careful not to allocate memory as RWX.  Instead, we opt for RW and then switch RX.  But Beacon's default reflective loader actually undoes this hard work.

Here we have injected Beacon shellcode into notepad.exe.

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/rwx/beacon.png)

If we inspect the memory regions in this process, we'll see the following:
  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/rwx/notepad-ph.png)

  

The two highlighted lines are the ones of interest.  The RX region is the one we allocated in our injector and contains Beacon's reflective loader.  The RWX region is where the actual Beacon payload is running.  So there are two issues here.

1.  We've got a dangling memory region that we don't need anymore.
2.  Beacon's RWX region is an OPSEC concern that we don't want.

Both can be fixed in Cobalt Strike's malleable C2 profile.

It's important to understand that the reflective loader is performing its own style of injection within the process.  It will allocate a block of memory, copy Beacon into it, and executes.  These behaviours are controlled via the `stage` malleable C2 block.

The first option is `allocator`, which controls the API used to allocate the memory region.  By default, **HeapAlloc** is used.  If you wish, this can be changed to **MapViewOfFile** or **VirtualAlloc**.  This doesn't change anything in regards to memory permissions, but good to know if you suspect the reflective loader is being detected due to this API call.

To prevent the use of RWX permissions, set `userwx` to **false**.  This will tell the reflective loader to allocate as RW and then flip to RX, the same as we've been doing in our injectors.

Finally, to clean up the memory region associated with the reflective loader, set `cleanup` to **true**.

```json
stage {
        set userwx "false";
        set cleanup "true";
}
```

If we generate and inject new shellcode with this profile we'll see Beacon split across different regions, each with correct permissions.  The header (RW), the main Beacon (RX) and everything else (RW).  

![](https://rto2-assets.s3.eu-west-2.amazonaws.com/evasion/rwx/rx-only.png)