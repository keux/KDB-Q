# Practical Guidance - Advanced IPC

* Deferred Synchronous Communication
* Deferred Response -30!
* Standard Tick Architecture and IPC

##  Deferred Synchronous Communication
Previously we have covered the two main types of IPC messages in q – sync and async. There is a third method of communication between two q processes, which is much less common but can be very useful in specific circumstances, called deferred sync requests.

This involves the sending of an async request, which has with the request embedded an async call back to the sending process (aka "a callback"). This leaves the sending process free to continue other operations while awaiting the response, unlike with standard synchronous sending.

Deferred sync requests allow a time consuming query to be sent to another q process, and allows the remote process to choose when to respond. The client process will wait for the result, but the remote process could (for example) choose to store the request somewhere, deal with other requests ahead of it, and then send the client’s result later.

To use deferred sync message, the client first sends the request to the remote server asynchronously:

```
q)neg[handle] "request goes here"
```
Then immediately blocks the handle by calling it as if it was calling a function with no arguments:

```
q)handle[]   //the process will wait until a return is received
```

On the server, it must be ensured that the the call-back handle (```.z.w```) is used to send an async response back to the client.

A trivial example would be:

```
q)neg[handle]"(neg .z.w) 1+1";handle[]
2
```
Of course this case is too trivial to be useful – in this case it makes more sense to do a sync query, since the client is effectively blocking itself until the remote server has responded.

## Deferred Response 
Ideally, for concurrency, all messaging would be async. However, sync messaging is a convenient paradigm for client apps. Hence -30!x was added as a feature in V3.6, allowing processing of a sync message to be ‘suspended’ to allow other messages to be processed prior to sending a response message.

You can use -30!(::) at any place in the execution path of ```.z.pg```, start up some work, allow .z.pg to complete without sending a response, and then when the workers complete the task, send the response explicitly.

There are some examples here of it in use within ```.z.pg``` when defining gateways.

## Standard Tick Architecture and IPC
The distinct between passing by reference and passing by value is the basis for most kdb+/q tick capture architectures.

Recap: 
```
    h (myFunc;1;2)   //calling local defintion myFunc:{x-y}
    -1

    h (`myFunc;1;2)  //calling remote definition myFunc:{x+y}
    3 
```
This distinction actually forms the basis for the standard tick messaging used in most kdb+/q system.

Messages are passed around the system in the form of:

  ```
  (`upd;<tablename>;<data>)
  ```
Since the function to be performed (upd) is passed using the symbol reference, this means each process in the system can choose how to define the action to be performed.

In the tickerplant for example this would usually be logging and message distribution while for the real-time database it would normally be to insert the provided data into the named table.

The worked examples at the end of the IPC notebook give some concrete examples of standard Tick Architecture behaviour using event handlers.


## Further Reading
* [Deferred Response with kdb+](https://kx.com/blog/kdb-q-insights-deferred-response/)
* [Permissions with kdb+](https://code.kx.com/q/wp/permissions/)
* [KDB+ and WebSockets](https://code.kx.com/q/wp/websockets/) 
* [Socket sharding with kdb+ and Linux](https://code.kx.com/q/wp/socket-sharding/)