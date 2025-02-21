# Advanced IPC Exercises

Before we start the Quiz, let's launch a remote q process that we will start with the `quizScript.q`
q file found in IPC.Scripts: 
```q
system["q  -p 5092 >/dev/null 2>&1 &"];
/system["nohup q  -p 5060 >/dev/null 2>&1 &"];
```
The following questions require a connection to this process. The current process can be considered 
to be a "Client" connection, and the remote process can be considered to be the "Server" process. 

## Password Validation Handler: ```.z.pw```

### Exercise 1
Connect to this newly started process, and store the connection handle in a variable called `h`.
Using this handle retrieve the IP address of the Server process
```q
h:hopen 5092;
```


### Exercise 2
Define a empty table on the Server process called .perm.users that has two columns - username (sym) and password (string). Key on the username column,
```q
h".perm.users:([username: `$()] password:())";
```


### Exercise 3 
Create a utility function called .perm.addUser on the Server process that takes 2 parameters [usr;pwd] and will add details to .perm.users

```q
h".perm.addUser:{[usr;pwd] `.perm.users upsert (usr;pwd)}";
```
Use this utility function add the below users with associated password credentials:
| username | password     |
|----------|--------------|
| harry    | wizard31     |
| ron      | scabbers01   |
| hermione | timeturner19 |

```q
{h(`.perm.addUser;x;y)}'[(`harry;`ron;`hermione);("wizard31";"scabbers01";"timeturner19")];
h".perm.users";
```


### Exercise 4
Create a utility function called .perm.removeUser on the Server process that takes 1 parameter [usr] and will remove users from .perm.users
```q
h".perm.removeUser:{[usr] delete from `.perm.users where username = usr}";
```
Now use this utility function to  remove Ron from the list of users:
```q
h(`.perm.removeUser;`ron);
h".perm.users";
```

### Exercise 5
Define ```.z.pw``` on the Server process that grants or denies access based on whether the user and password details match those stored in .perm.users.

Desired outcome test cases:
* ```hermioneHandle:hopen `::5092:hermione ``` should get access denied
* ```hermioneHandle:hopen `::5092:hermione:guess123 ``` should get access denied
* ```hermioneHandle:hopen `::5092:hermione:timeturner19``` should get access granted
```q
h".z.pw:{[usr;pwd] pwd~.perm.users[usr][`password]}";
hermioneHandle:hopen `::5092:hermione;
hermioneHandle:hopen `::5092:hermione:guess123;
hermioneHandle:hopen `::5092:hermione:timeturner19;
```

## Explunge (=reset): ```\x .z.p*```
### Exercise 6
Expunge the definition of ```.z.pw``` such that all users attempting to connect will be granted access.
```q
h"\\x .z.pw";
```

## Opening Event Handler: ```.z.po```

### Exercise 7
Define a new empty table on our Server process called .perm.accessLog with the following schema:
* [user (symbol)] handle (int), time (timestamp), state (symbol)
```q
h".perm.accessLog:([user: `$()] handle:`int$(); time:`timestamp$(); state:`$())";
h"meta .perm.accessLog";
```

### Exercise 8
Define ```.z.po``` on our Server process such that it will add to .perm.accessLog and maintain a history of all opened connections i.e. state=open
```q
h".z.po:{`.perm.accessLog upsert (.z.u; x; .z.p; `open)}";
```
Test your event handlers is doing what it should be by opening a few connections
```q
h".perm.accessLog"  // empty before anyone new has connected
```
```q
hermioneHandle:hopen `::5092:hermione
harrysHandle:hopen `::5092:harry
ronHandle:hopen `::5092:ron
```
```q
h".perm.accessLog"  // table should now contain 3 entries
```

### Exercise 9
Define ```.z.pc``` on our Server process such that it will add to .perm.accessLog and maintain a history of all closed connections i.e. state=close .
```q
h".z.po:{`.perm.accessLog upsert `user`handle`time`state!(.z.u;x;.z.p;`close)}";
```
Test your event handlers is doing what it should be by closing a few connections
```q
h".perm.accessLog"  // no change yet
```
```q
hclose hermioneHandle
```
```q
h".perm.accessLog"  // table should now contain additional entry showing closed connections
// note user will client user rather than hermione as we are running hclose as ourselevs 
// in a normal setup the user connecting via hopen and then subsequently closing the handle would be the same user
```


### Exercise 10
Define a new table on our Server process called .perm.queryLog with the following schema:
* user (symbol), time (timestamp), query(string), sync (boolean)
```q
h".perm.queryLog:(user:`$(); time:`timestamp$(); query:(); sync:`boolean$())"
```

### Exercise 11
Define ```.z.pg``` on our Server process such that any incoming synchronous request is:
* added to .perm.queryLog
* sync=1b 
* ```.Q.s1``` used  to format incoming query
* then executed

*.z.pg should take one parameter which is query*

**Tip:** You may run into the issue were you defined ```.z.pg``` erroneously on first few attempts which blocks subsequent calls to the Server. Because we can't access our Server process directly we might want to test table and adding to it with a local definiton first on Client to ensure function works before defining it on Server.

But if this happens you can revert ```.z.pg``` on the Server process by running expunge asynchronously:
```q
(neg h)"\\x .z.pg"
```

```q
h".z.pg:{[query] `.perm.queryLog upsert `user`time`query`sync!(.z.u;.z.p;.Q.s1 query;1b); value query}"
```
Test your function works by running a few queries and checking they get added to the query log
```q
harrysHandle"til 100"
ronHandle"3*3"
h".perm.queryLog"       // should see .perm.queryLog is now logging all incoming queries before they are executed.
```

### Exercise 12
Define ```.z.ps``` on our Server process such that any incoming asynchronous request is:
* added to .perm.queryLog
* sync=0b 
* ```.Q.s1``` used  to format incoming query
* not executed
* attempt to pass message returned to calling process saying "No asynchronous connections allowed on this process"
```q
h".z.ps:{[query] `.perm.queryLog upsert `user`time`query`sync!(.z.u;.z.p;.Q.s1 query;0b); 0N!\"No asynchronous connections allowed on this process\"}"
```
Test your function works by running a few async queries and checking they get added to the query log, we should not see message returned however
```q
(neg harrysHandle)"sum 8 100 5"     // no message returned even though we specified one
(neg ronHandle)"3*3"
h"`time xdesc .perm.queryLog"       // should see .perm.queryLog is now logging all asynch queries
```

### Exercise 13
Create a utility function called sendMail on the Server process that takes 2 parameters and will return a message to the sender from the respondent. 
* start by expunging .z.ps from the Server process or these calls will be blocked ```\x .z.ps```
* save a function on both Client and Server processes ```echo:{0N!x;}```
* send a message from the Client process, the Server should recieve hedwig
* when the Server process recieves the message, the Client process should recieve pigwidgeon
```q
h"\\x .z.ps";

h"echo:{0N!x;}";
echo:{0N!x;};

h"sendMail:{[query; callbackfunc] echo query; h:.z.w; neg[h](callbackfunc;`pigwidgeon)}";
```
Test your function works by running a few async queries, checking that your Server process recieves the name of the message you sent and your Client recieves the name of the message sent from the Server. 
```q
(neg h)(`sendMail;`hedwig;`echo)
```