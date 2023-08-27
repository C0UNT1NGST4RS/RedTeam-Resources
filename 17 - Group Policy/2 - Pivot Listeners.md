This is a good time to segway to talk about Cobalt Strike's Pivot Listener.

This is another type of P2P listener that (currently only) uses TCP, but it works in the opposite direction to the regular TCP listener. When you spawn a Beacon payload that uses the TCP listener, that Beacon acts as a TCP server and waits for an incoming connection from an existing Beacon (TCP client).

Pivot Listeners are not created via the Listeners menu, but are bound to individual Beacons. This existing Beacon will bind a port and listen for incoming connections (acting as the TCP server), and a Beacon payload that uses the Pivot Listener will act as the TCP client.

Why is this useful?

In scenarios such as GPO abuse, you don't know when the target will actually execute your payload and therefore when you need issue the `connect` command. When a Beacon checks in over a Pivot listener, it will appear in the UI immediately without having to manually connect to it.

To start a Pivot Listener on an existing Beacon, right-click it and select **Pivoting > Listener**.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/pivot-listener/new-pivot-listener.png)

  

Once started, your selected port will be bound on that machine.

```
beacon> run netstat -anp tcp

Active Connections

  Proto  Local Address          Foreign Address        State
  TCP    0.0.0.0:4444           0.0.0.0:0              LISTENING
```

Like other P2P listeners, you need to consider whether this port will be reachable (i.e. the Windows Firewall may block it) and act accordingly.  A well configured and locked-down firewall can significantly increase the difficultly of lateral movement.

-   If port 445 is closed on the target, we can't use SMB listeners.
-   If the target firewall doesn't allow arbitrary ports inbound, we can't use TCP listeners.
-   If the current machine doesn't allow arbitrary ports inbound, we can't use Pivot listeners.

It may become necessary to strategically open ports on the Windows firewall to facilitate lateral movement.  This can be done with the built-in `netsh` utility. To add an allow rule:

```
netsh advfirewall firewall add rule name="Allow 4444" dir=in action=allow protocol=TCP localport=4444
```


To remove that rule:

```
netsh advfirewall firewall delete rule name="Allow 4444" protocol=TCP localport=4444
```

  

You can generate payloads for the pivot listener in exactly the same way as other listeners. When executed on a target, you should see the Beacon appear automatically. You will also notice the arrow is pointing the opposite direction compared to a normal TCP Beacon.

  

![](https://rto-assets.s3.eu-west-2.amazonaws.com/pivot-listener/pivot.png)

  

Pivot listeners can be stopped from the regular Listeners menu.