# Advanced Inter-Process Communication

##### Learning Outcomes

To understand:

* .z Namespace
* Event Handler for Password Validation - .z.pw
* Event Handlers for Connection Open/Close - .z.po .z.pc
* Event Handlers for Sync/Async Messages - .z.pg .z.ps
* Event Handlers for HTTP - .z.ph
* Flushing Messages

# Introduction

Inter process communication is essential and a fundamental part of real-world applications using 
kdb+. Kdb+/q conveniently has a built in method of connecting to other kdb+/q processes to retrieve 
and/or manipulate data.

The basics of [IPC](https://code.kx.com/q/learn/startingkdb/ipc/) have already been covered in our Fundamentals courses. This module will cover more advanced aspects of IPC namely event handlers and the ```.z``` namespace.

#### [Recap](https://code.kx.com/q/basics/ipc/)

|              |                                                          |
|--------------|----------------------------------------------------------|
| \p -p        | listen to a port                                         |
| hopen hclose | open/close connection                                    |
| .z.W         | IPC handles with number of bytes waiting in output queue |
| h            | synchronous messaging - waits for result to be returned  |
| neg h        | asynchronous messaging - does not wait for result        |

Valid ```hopen``` declarations:
```
hopen `:<computer host/IP>:<port number>:<username>:<password>
hopen `:localhost:<port number>   //localhost is understood
hopen `::<port number>            //this is valid - localhost is assumed when a host isn't provided
hopen <port number>               //this is also valid - assumes localhost
```

# .z Namespace

The [`.z` namespace](https://code.kx.com/q/ref/dotz/)  contains all environment variables as well as hooks for callbacks.

By now we should be comfortable opening and closing connections. The next step is tracking and managing these connections, this is where functions in the  ```.z``` namespace come into play. 

## .z Environment Variables
We have already seen many of these before like:
```q
.z.d // date shortcut
```
```q
.z.K // kdb release version
```
As well as these there are also several variables available when any message handler is invoked. 
|      |                                   |
|------|-----------------------------------|
| .z.a | IP address of calling process     |
| .z.u | Username of calling process       |
| .z.w | Integer handle of calling process |

The variables are important to allow us to determine the origin of the incoming connection, and in the case of .z.w to enable more advanced communication than standard sync or async.

```q
//when called outside of an event handler these variables are local
.z.a              //IP address 
```
```q
.z.u              //user on the box
```


In the same way that the client will have its own handle h to talk to server - the server actually has its own handle set up to talk back to the client ```.z.w```.
```q
.z.w    // because we are in Kx Developer session we are already connecting via IPC
// this would be 0i on a session on regular console
```
```.z.w``` will already be there for you to use without and additional setup required.

Observe in the following example how we can use ```.z.w``` to get the integer handle of the calling Client process to 'talk back' to Client.
![](./images/ipc0.gif)

**DIY time!**
Callback functions are very useful when we are using asynchronous calls. If we want to receive a response from the client when we are sending the message asynchronously, we can define a callback function.
Using the script callbackExample.q in AdvancedIPC.Scripts module, we can see how to define our new callback function

# .z Event Handlers

q has built in event handlers; that is internal functions which are specifically called when the different types of IPC are performed. 

They all live in the ```.z namespace``` and can be modified for specific uses. The ones we will be focusing on today are some of the most commonly used:

|       |                     |
|-------|---------------------|
| .z.pw | Password validation |
| .z.po | Port open           |
| .z.pc | Port close          |
| .z.pg | Synchronous get     |
| .z.ps | Asynchronous set    |
| .z.ph | HTTP                |

They are extremely powerful hooks that are highly customizable.

Event Handlers are not defined by default in kdb+ (except ```.z.ph```). User can override these special functions to handle authentication and client calls.


## Event handler for password validation - ```.z.pw```
### User Validation - ```.z.pw```
```.z.pw``` is a hook for checking validity of connection user. 

```.z.pw``` is evaluated after the ```-u/-U``` checks, and before ```.z.po``` when opening a new connection to a kdb+ session.


* The parameters to ```.z.pw``` are the username (symbol) and password (string) supplied by the incoming call.
* It should return a boolean result ```1b/0b```
* If ```.z.pw``` returns ```0b``` the task attempting to establish the connection will get an ```'access``` error.
```
.z.pw:{[user;pswd]1b}
```

We can check that ```.z.pw``` is indeed undefined on our Server process:
```q
h:hopen 5091       // opening handle to Server process
```
```q
h".z.pw"        // yep it is undefined 
```
Lets define a table .perm.users on our Server process where we will store the users allowed to authenticate to our Server process.

```q
h".perm.users:([user:`mary`john`ann ] password:(\"pwd\";\"pwd\";\"pwd\"))"
```
```q
h".perm.users"
```
Before we modify ```.z.pw``` lets check that we can connect without any password credentials.
```q
maryHandle:hopen `::5091:mary
```
Now lets modify ```.z.pw``` to do a simple check on if the incoming user is authenticated.
```q
h".z.pw:{[user;pswd] $[pswd~.perm.users[user][`password]; 1b; 0b]}"
```
And lets retry with same details as before:
```q
maryHandle:hopen `::5091:mary // `access error
```
Now we get an ```'access``` error. We now need to add the correct password in order to authenticate succesfully.
```q
maryHandle:hopen `::5091:mary:pwd // once correct password added we can connect ok
```
Lets change the password to something else and see what happens:
```q
maryHandle:hopen `::5091:mary:pwd2 // get `access error again with incorrect password
```


#### Expunge an event handler - ```\x```

What if we want to undo the definition of ```.z.pw``` and allow all connecting users to connect again?

We can use ```\x``` to expunge to reset the ```.z.pw``` event handle.

Format of this is ```\x eventHandle```.

Note we will need to run this command as using Mary's handle.
```q
maryHandle"\\x .z.pw"       // additional backslach required to escape quotes
```
And retrying our details again:
```q
maryHandle:hopen `::5091:mary:pwd2 // all good now as .z.pw definition has been reverted
```

As .z.pw is simply a function it can be used to implement rules stored locally like checking against a table we maintain of authenticated users on our server; or we can go out to external resources like an LDAP or Kerberos.

**Quiz Time!**

*Try Exercises 1-6 to test your understanding of ```.z.pw``` and have a go of implementing it for yourself!*

## Event Handlers for Opening & Closing Connections - ```.z.po``` & ```.z.pc```

The event handlers ```.z.po``` (port open) and ```.z.pc``` (port close) are invoked when a remote process opens or closes a connection to our process. 

The sole argument to both of these functions is the handle of the connection that has been opened / closed.

In the following example we modify ```.z.po``` and ```.z.pc``` to see how we can output a simple message every time a new connection is opened or closed.
![](./images/ipc1.gif)

### Opening Connection Handler - ```.z.po```
The ```.z.po``` event handle is ran immediately after ```.z.pw``` provided it is defined and user has successfully authenticated.

More often we may want to capture information about the process connecting rather than just displaying them.

Lets say we want to capture all handles that connect to our process and store it in a list.

Lets define empty list on Server process called .ipc.handles:
```q
h".ipc.handles:()"
```
Now lets define ```.z.po``` to say add all new incoming handles to list:
```q
h".z.po:{.ipc.handles,:x}"
```
Lets check contents of .ipc.handles before we open a new connection:
```q
h".ipc.handles"     // can see its still empty
```
Now open another handle to process and recheck:
```q
h:hopen 5091
h".ipc.handles"     // can see it contains a new handle
```
If you continue to rerun above cell we can see everytime a new connection is opened a new entry is added to our list.

Let's say we want to capture more information from the process as part of our action, we can use the ```.z.w``` variable to do this - when called from within an event handler like ```.z.po```, .z.w works as a handle back to the calling process.

Define a varible called ID on Client Process
```q
ID:`ClientA
```
Define a new table called .ipc.connections on our Server process.
```q
h".ipc.connections: ([handle:()];time: ();user:();id:();state:())"
```
Redefine our open connection handler ```.z.po``` to update the table with the connection details.
```q
h".z.po:{ `.ipc.connections insert (x;.z.p;.z.u;.z.w `ID;`open)}"
//.z.u is the username of the calling process
//.z.p is the time when connection was opening 
// using .z.w we are able to reference the variable ID on Client process 
// state will be `open to help us differentiate open verses closed connections
```
```q
h".ipc.connections"     // empty right now
```
Lets open a connection and see what happens
```q
h:hopen 5091
h".ipc.connections"
```
Rerunning this cell a few times you can see how the table is getting added to with each open connection.

**Quiz Time!**

*Try Exercises 7-8 to test your understanding of ```.z.po``` and have a go of implementing it for yourself!*

### Closing Connection Handler - ```.z.pc```
Lets implement similar logic for whenever a handle is closed.
```q
h".z.pc"        // again like .z.po, .z.pc is not defined by default
```
We can define ```.z.pc``` to take some action upon a connection closing - we will change the state from open to closed.
```q
h".z.pc:{`.ipc.connections upsert `handle`time`state!(x;.z.p;`close)}"
```
Checking .ipc.connections:
```q
h".ipc.connections"     // no change yet
```
Now lets close one of our open connections.
```q
h2:last (key .z.W )except h        //use .z.W to check open connection handles and get last handle opened that isn't h
hclose h2               // close connection
h".ipc.connections"     // check state has changed
```
Replaying this last cell we can see how ```.z.pc``` can be used to manage state.

We could also modify ```.z.pc``` to delete a connection upon closing.
```q
h".z.pc:{delete from `.ipc.connections where handle = x}"
```
```q
h2:last (key .z.W )except h        //use .z.W to check open connection handles and get last handle opened that isn't h
hclose h2               // close connection
h".ipc.connections"     // can see there is one less entry in table
```

**Quiz Time!**

*Try Exercise 9 to test your understanding of ```.z.po``` and have a go of implementing it for yourself!*

## Event Handlers for sync/async messages - ```.z.pg``` & ```.z.ps```
The event handlers ```.z.pg``` (port get) and ```.z.ps``` (port set) handle synchronous and asynchronous calls respectively. 

They are different to ```.z.po``` and ```.z.pc``` in that the logic you place in them actually affects the outcome of the event itself, rather than acting as a side effect.

### Synchronous Message Handler - ```.z.pg```
Port Get Handler that is invoked when any synchronous call is made to kdb+/q process.

Can be overridden and modified to:
* Restrict access to certain users 
* Log incoming calls 
* Debugging – to see exactly what calls are happening on a process

Note when defining .z.pg it is important to return a result, usually done by executing a value on the input.
```q
h".z.pg"  // not defined by default
```

#### Restrict access to certain users
Lets define a table .perm.users on our Server process where we will store the users allowed to authenticate to our Server process.
```q
h".perm.users:([user:`mary`john`ann ] class:`basicUser`superUser`basicUser; password:(\"pwd\";\"pwd\";\"pwd\"))"
```
```q
h".perm.users"
```
Let's see if our users John & Mary have access to connect.
```q
johnHandle:hopen `::5091:john:pwd
```
```q
maryHandle:hopen `::5091:mary:pwd
```
```q
johnHandle"2+2"     // all good for John
```
```q
maryHandle"2+2"     // all good for Mary
```
Can see we have permissions to run as both John & Mary.

Now lets define ```.z.pg``` to allow any superUser connecting to run their query but prevents basicUsers from doing so.
```q
h".z.pg:{[query]class:.perm.users[.z.u][`class]; $[class~`superUser;value query;\"No Permissions\"]}"
```
Lets re-run as John & Mary again:
```q
johnHandle"2+2"     // all ok still
```
```q
maryHandle"2+2"     // not ok anymore
```
As Mary only has the class basicUser she is not allowed to run her query on the process. 


As John is the only superUser this means our old handle ```h``` can no longer connect.
```q
h"2+2"
```

Lets run expunge again, note we will need to run this command as superUser John as no one else has access to run queries right now!
```q
johnHandle"\\x .z.pg"       // additional backslach required to escape quotes
```
And retrying our other handles again:
```q
h"2+2"      // Phew! access to run queries using h has been restored
```
```q
maryHandle"2+2"     // and the same for our basicUser Mary
```

This is a very basic example of how ```.z.pg``` can be modified to restrict access to incoming queries and how to reset it using ```\x```.

**Quiz Time!**

*Try Exercises 10-11 to test your understanding of ```.z.pg``` and have a go of implementing it for yourself!*

### Asynchronous Message Handler - ```.z.ps```
The asynchronous message handler is similar to the synchronous message handler, the only difference being that .z.ps does not return a result when invoked.

Even if the function explicitly defines a return, it won't be returned to the calling process.

Let's take a look at how ```.z.ps``` diffes to ```.z.pg``` by setting both to print message back to Client process.
```q
// define .z.pg on Server Process
h".z.pg:{\"New synchronous message: \",.Q.s1 x}";

// define .z.ps on Server Process
h".z.ps:{\"New asynchronous message: \",.Q.s1 x}";
```
```q
h"2+2"      // Message printed out to Console for synchronous message
```
```q
(neg h)"2+2"    // No message printed out to Console for asynchronous message
```
We can see that even though we have set ```.z.ps``` to explicitly return a message it does not get returned.

Let's revert back to default for both:
```q
(neg h)"\\x .z.pg" 
(neg h)"\\x .z.ps" 
```
```q
h"2+2"      // back to normal for synchronous
```
```q
(neg h)"2+2"    // and for asynchronous no noticable change but we have reset
```
The exact same principles hold true for asynchronous calls and ```.z.ps``` as do above for ```.z.pg```, the only difference being that it doesn't matter what you return from ```.z.ps```, as the caller will not be waiting for the result.

#### Firewalling with ```.z.ps```
When making a process open for connections, it is important to consider how do we put appropriate firewalls in place to prevent malicious activity. Within kdb+/q, firewalling is linked to the ```.z namespace```, and effectively means disabling them from outside use by assignment. 

For example assigning ```.z.ps:{}``` will prevent any asynchronous requests being executed as the handle has been defined to not execute any asynchronous messages.

**Quiz Time!**

*Try Exercise 12 to test your understanding of ```.z.ps``` and have a go of implementing it for yourself!*

## Event handlers for HTTP - ```.z.ph```

So far we have shown how to restrict and control access from a client process that connects via IPC. However, it is also possible to connect to a kdb+ process via HTTP.

While ```.z.pw``` works for both IPC and HTTP, HTTP queries are not routed through the ```.z.pg``` handler – they are handled by ```.z.ph```.

```.z.ph``` is the only message handler with a default definition. It is responsible for composing the HTML webpage, executing the query, and formatting the results into a HTML table.

We can take a look at the default definition here:
```q
h".z.ph"
```
```.z.ph``` can be redefined entirely with a custom web server if required. However, it is more usual to supplement the behaviour of ```.z.ph```. 

For example, the following could be used to log and execute browser queries if the function log_query is defined already:
```
q).orig.zph:.z.ph 
q).z.ph:{log_query[x]; .orig.zph[x]}
```

## Flushing Messages
Sometimes it is useful to send a large number of aysnc messages, but then to block until they have all been sent.

This can be achieved through using async flush – invoked as:
```
q)neg[h][]
//or 
q)neg[h](::)
```

If you need confirmation that the remote end has received and processed the async messages, chase them with a sync request, e.g:
```
h""
```
The remote end will process the messages on a socket in the order that they are sent.


## Callbacks
The construct of an asynchronous remote call with callback is not built into interprocess communication (IPC) syntax in q, but it is not difficult to implement. We first need to define a simple function on both client and server processes.
```
q)echo:{0N!x;}
```
As we will be sending a message from the client processes which contains the name of a function to be called from the server process, we must also define another function on the server side. 
```
q)proc:{echo x; h:.z.w; (neg h) (y; 43)}
```
This function will run the echo function on its own process whilst also sending a return message. On the client process we can send an synchronous call to the server process with the name of the proc function with an argument and the callback function echo.
```
q)(neg h) (`proc; 42; `echo)
```
The echo function will therefore run on both client and server processes and we will get a clear indication that the server has recieved the message and is able to send a message back to the server. This is a great way of checking for busy processes.


## Further Reading
* [Deferred Response with kdb+](https://kx.com/blog/kdb-q-insights-deferred-response/)
* [Permissions with kdb+](https://code.kx.com/q/wp/permissions/)
* [KDB+ and WebSockets](https://code.kx.com/q/wp/websockets/) 
* [Socket sharding with kdb+ and Linux](https://code.kx.com/q/wp/socket-sharding/)
* [KDB+ and Callbacks](https://code.kx.com/q/kb/callbacks/)
