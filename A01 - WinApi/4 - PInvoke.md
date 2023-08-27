Platform Invoke (P/Invoke) allows us to access structs and functions present in unmanaged libraries from our managed code.

Applications and libraries written in C/C++ compile to machine code, and are examples of unmanaged code.  Programmers must manage aspects like memory management manually - e.g. whenever they allocate memory, they must remember to free it.  In contrast, managed code runs on a CLR, or Common Language Runtime.  Languages such as C# compile to an Intermediate Language (IL) which the CLR later converts to machine code during runtime.  The CLR also handles aspects like garbage collection and various runtime checking, hence the name managed code.

Why do we even need P/Invoke?  Let's use .NET as an example.  

The .NET runtime already utilises P/Invoke under the hood, and provides us with abstractions that run on top.  For instance, to start a process in .NET we can use the `Start` method in the `System.Diagnostics.Process` class.  If we trace this method in the runtime, we'll see that it uses P/Invoke to call the CreateProcess API.  However, it doesn't provide a mean that allows us to customise the data being passed into the STARTUPINFO struct; and this prevents us from being able to do things like start the process in a suspended state.

There are other WinAPIs that are useful for us (such as VirtualAllocEx, WriteProcessMemory, CreateRemoteThread, etc) that are not exposed at all in .NET; and the only way we can access them is to P/Invoke manually within our code.

Other languages that support P/Invoke also open a wealth of opportunity for attackers.  For instance, we can use P/Invoke in VBA which lends a certain potency to malicious Office documents.