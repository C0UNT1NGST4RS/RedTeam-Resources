Many of Beacon's indicators are controllable via malleable C2 profiles, including network and in-memory artifacts. This section will focus on network artifacts. We know Beacon can communicate over HTTP(S), but what does that traffic look like on the wire? What are the URLs? Does it use GET, POST, other? What headers or cookies does it have? What about the body? All of these elements can be controlled.

Raphael has several example profiles [here](https://github.com/Cobalt-Strike/Malleable-C2-Profiles). Let's use the [webbug.profile](https://github.com/Cobalt-Strike/Malleable-C2-Profiles/blob/master/normal/webbug.profile) to explain these directives.

First we have an `http-get` block - this defines the indicators for an HTTP GET request.

`set uri` specifies the URI that the client and server will use. Usually, if the server receives a transaction on a URI that does not match its profile, it will automatically return a 404.

Within the `client` block, we can add additional parameters that appear after the URI, these are simple key and values. `parameter "utmac" "UA-2202604-2";` would be added to the URI like: `/__utm.gif?utmac=UA-2202604-2`.

Next is the `metadata` block. When Beacon talks to the Team Server, it has to be identified in some way. This metadata can be transformed and hidden within the HTTP request. First, we specify the transform - possible values inclue `netbios`, `base64` and `mask` (which is a XOR mask). Then we can append and/or prepend string data. Finally, we specify where in the transaction it will be - this can be a URI `parameter`, a `header` or in the body.

In the webbug profile, the metadata would look something like this: `__utma=GMGLGGGKGKGEHDGGGMGKGMHDGEGHGGGI`.

Next comes the `server` block which defines how the response from the team server will look. Any headers are provided first. The `output` block dictates how the data being sent by the Team Server will be transformed. At this point, it should be fairly clear. Arbitrary data can be appended or prepended before being terminated by the `print` statement.

The `http-post` block is exactly the same but for HTTP POST transactions.

Customising the HTTP Beacon traffic in this way can be used to simulate specific threats. Beacon traffic can be dressed to look like other toolsets and malware.