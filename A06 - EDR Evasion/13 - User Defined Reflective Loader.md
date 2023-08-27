The User Defined Reflective Loader (UDRL) is a kit which allows you to customise the reflective loader used in Beacon workflows.  Reflective loading is used all over the place in Cobalt Strike, including:

-   Loading the reflective Beacon package in an artifact/payload.
-   Post-ex commands such as dllinject, execute-assembly, powerpick, keylogger, screenshot, etc

Beacon has always had a "default" reflective loader and various configuration options such as `userwx` and `cleanup` are exposed via Malleable C2 Profiles.  The creation of this kit provides complete customisation over the reflective loader - more so than with a C2 profile.

However, it's important to note that a custom UDRL will take priority over C2 profile directives.  For instance, if you have set userwx `"false"` in your profile but allocate RWX in your UDRL, then you will see RWX regions in memory.  Changes to the Artifact Kit are still important, because an artifact is responsible for getting the reflective loader into memory and executing it.  The reflective loader will then load and execute the Beacon package, so it's a multi-step process.

The UDRL is based on [Stephen Fewer](https://twitter.com/stephenfewer)'s original [ReflectiveDLLInject](https://github.com/stephenfewer/ReflectiveDLLInjection) project, which is provided under a 3 clause BSD license.  The upshot is that this allows anybody to publish their own Cobalt Strike compatible reflective loaders - unlike the Artifact Kit, which is provided under the same end user license agreement as Cobalt Strike (i.e. redistributions are not allowed).  This is why you can find public UDRL projects, but not public artifact kit projects.

  

Some public loaders include:

-   [https://github.com/boku7/BokuLoader](https://github.com/boku7/BokuLoader)
-   [https://github.com/Cracked5pider/KaynStrike](https://github.com/Cracked5pider/KaynStrike)
-   [https://github.com/mgeeky/ElusiveMice](https://github.com/mgeeky/ElusiveMice)
