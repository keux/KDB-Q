# Parallel Processing
## Learning objectives
1. What is parallel processing?
2. How to use parallel processing in kdb+?
    1. Secondary Threads ( ```-s``` )
    2. Parallel Each ( ```peach``` )
    3. Parallel Cut ( ```.Q.fc``` )
3. Considerations when using parallel processing
    1. Query computation vs Memory overhead
    2. Workload balancing
    3. Memory Usage
4. When should I use parallel processing?
    1. Query Optimization
    2. Database Maintenance
    3. Writing/Reading Flat Files


# Introduction
Parallel processing is running two or more processes simulatenously and then dividing up tasks between the processors in order to decrease overall execution time. It is commonly used to perform complex tasks and computations.

# When can we use parallel processing?

Any system that has more than one core can perform parallel processing, as well as multi-core processors which are commonly found on computers today.

We can check the number of cores on our virtual machine that is hosting our q process using the system variable ```.z.c```.

```q
.z.c
```
We can see this machine has 2 cores - so we know we can implement parallel processing on this machine!

The number of cores is also displayed in the banner when you launch a new q process. For example launching q  on your local machine you might see 8 cores as follows:
```
KDB+ 4.0 2020.07.15 Copyright (C) 1993-2020 Kx Systems
w64/ 8(8)core 7998MB mwoods lptp1291 172.24.0.1 EXPIRE 2022.02.18 mwoods@kx.com #62736
```

### Environmental Considerations for this module :
* If you are running this module locally you will most likely have more cores than this Virtual Machine - this could impact the results of certain examples.
* This VM is running kdb+ v3.6
```q
.z.K
```
* If you are running a different version you will likely experience different results - for e.g. v4.0 introduces multithreaded primitives - will be discussed later
* For the best training experience we would recommend using v3.6 for this module and associated exercises 


# How to use parallel processing in kdb+?

## Secondary Threads
To execute in parallel, first we need to start kdb+ with multiple secondary threads, using ```-s``` in the command line.

*Note older terminology for secondary threads was slaves which you may find reference to elsewhere - they mean the same thing!*

### ```-s``` (command line option)
```
// Starting q process with 2 secondary threads
C:\>q -s 2
KDB+ 4.0 2020.07.15 Copyright (C) 1993-2020 Kx Systems
w64/ 8(8)core 7998MB mwoods lptp1291 172.24.0.1 EXPIRE 2022.02.18 mwoods@kx.com #62736

q)\s
2i
```

### ```\s``` (number of secondary threads)
By default a q instance has no secondary threads- we can check the number of threads on this process using ```\s```, 0i is returned indicating no secondary threads:
```q
\s
```

Since V3.5, we can also use ```\s N```, where ```N``` is an integer, to set the number of secondary threads available for parallel procssesing up to the limit set by the ```-s``` command line option.

![](./images/parallel0.gif)

We can only use ```\s``` to adjust number of  threads up to the maximum specified on the command line.

As we are using an IDE on a Virtual Machine we will access a background process that has been started for us with multiple threads.
```q
h2:hopen `::5093     // opening a handle to background process with 2 secondary threads
```
```q
h2"\\s" / checking number of sec. threads = 2
```


Parallel execution can be explicitly invoked by using two built-in functions: ```peach``` and ```.Q.fc```.

**Quiz Time!**

*Try Exercises 1-3 to test your understanding so far.*


## Parallel Each ( ```peach``` )
Now we have access to a process running with multiple threads, the next step is to delegate tasks to these secondary threads for parallel execution.

We use parallel each ```':``` or the keyword ```peach``` to do this, notation is:
```
q)f peach x // execute function f on x over secondary threads
```
```peach``` bases distribution on indices.
E.g. For 2 threads work is split:
* Thread 1: 0, 2, 4…
* Thread 2: 1, 3, 5…

```peach``` will execute a function over multiple secondary processes, passing the arguments and results between the secondary processes and the main thread using IPC serialization.


### ```peach``` vs ```each``` :

* The result of ```peach``` will be exactly the same as the ```each``` keyword
* When no secondary threads are present, ```peach``` performs exactly like ```each```

Let's look at an example using a slow function that takes a significant time to execute.

This function which will create a list of floats of the same length as the argument and raise it to a specific power, then return the sum of the result.
```q
// each
h2"\\t {prd (x?1.0) xexp 1.7} each 2#10000000"
```
```q
// peach
h2"\\t {prd (x?1.0) xexp 1.7} peach 2#10000000"
```
We can see here ```peach``` is slightly quicker than ```each``` in this example.

**Quiz Time!**

*Try Exercises 3-10 to test your understanding of each verses peach.*

## Parallel Cut ( ```.Q.fc``` )
Parallel cut or ```.Q.fc``` distributes its load over multiple threads by cutting a vector into n number of segments, where n is the amount of cores available.

E.g. For 2 threads:
* Thread 1: 0, 1, 2
* Thread 2: 3, 4, 5

```.Q.fc``` vs. ```peach```

![](./images/peachDiagram.PNG)

```.Q.fc``` is used to carry out vector operations in parallel. 

```q
// lets's define vector of 10,000,000 floats
h2"vector0:10000000?1.0"
```
```q
// lets run a function that gets the exponent of each float
// first using peach
h2"\\t a0:{ x xexp 1.7 } peach  vector0"
```
```q
// secondly using parallel cut .Q.fc
h2"\\t a1:.Q.fc[{ x xexp 1.7 }] vector0"
```
We can see in this example ```.Q.fc``` is must faster than ```peach```- why?

* Thinking back to our diagram above for ```peach``` each float is passed seperately to the secondary threads
* The maximises data transfer overhead
* Whearas with ```.Q.fc``` the function is executed only twice over the two threads - once on each thread.

**Quiz Time!**

*Try Exercises 11-14 to test your understanding of .Q.fc verses peach.*

# Considerations when using parallel processing
## Query computation vs. Memory overhead

When considering whether to use parallel processing first we need to consider if the operation is computationally expensive enough to justify the messaging overhead. 

In the case of a very simple operation, using ```peach``` can be slower than just executing sequentially.

Let's take an example with a very simple computation, just doing a product:
```q
// each 
h2"\\t:1000 { x*1?10 } each 10?1.0"
```
```q
// peach
h2"\\t:1000 { x*1?10 } peach 10?1.0"
```

In this example ```peach``` is slower than just executing sequentially - this is because the overhead of passing data to the threads outweighs the paralleization benefits!

Let's increase the complexity and see what happens:
```q
// each
h2"\\t:100 {  sqrt (10000?x) xexp 1.7 } each 10?1.0"
```
```q
// peach
h2"\\t:100 {  sqrt (10000?x) xexp 1.7 } peach 10?1.0"
```
Adding the ```sqrt``` comuputation has meant now we see a reduction in time taken to execute our function.

We can estimate the serialization time and space of a q entity with ```\ts:100 -9!-8!entity```
```q
h2"\\t:100 -9!-8!1000000?1.0"
```
The parallel overhead from copying data to secondary threads can be estimated by using the [-8!](https://code.kx.com/q/basics/internal/#-8x-to-bytes) operator to serialize the argument and then de-serializing with the [-9!](https://code.kx.com/q/basics/internal/#-9x-from-bytes) operator.

A function applied with ```peach``` should do enough computation to outweigh the serialization overhead. 

## Workload Balancing

Work Distribution is very important when considering parallel processing. This is particularly important when the process is handling a large number of tasks of uneven size.

This distribution used in ```peach``` can be shown in q using the number of secondary processes and the modulus function ```mod```.

```q
// 6 input arguments over 2 cores
h2"(til 6) mod 2 "
```
```q
// 14 input arguments over 6 cores
h2"(til 14) mod 6 "
```
To demonstrate the effect of uneven distribution let's define a function f0 with an execution time proportional to the input argument:
```q
h2"f0: { x:7000*x;sum sqrt (x?1.0) xexp 1.7};"
```
```q
h2"\\t f0[100]"
```
```q
h2"\\t f0[1000]"
```
So increasing the size of the argument increases time for function to run.
#### ```each```
```q
// Single threaded - Balanced
h2"\\t f0 each 10 10 10 10 1000 1000 1000 1000"
```
```q
// Single threaded - Unbalanced
h2"\\t f0 each 10 1000 10 1000 10 1000 10 1000"
```
Performance is virtually the same here
#### ```peach```
```q
// Multithreaded - Unbalanced
h2"\\t f0 peach 10 1000 10 1000 10 1000 10 1000"
```
In this second case, there is little to no improvement from using ```peach```, in fact is has gotten worse.

This is because the fast and slow jobs alternate in the input list, which means all the slow tasks are assigned to a single thread.

The main process must wait for the second thread to execute all four slow jobs even though the first thread is finished.
```q
// Multithreaded - Balanced
h2"\\t f0 peach 10 10 10 10 1000 1000 1000 1000"
```
This case is a balanced distribution, with fast and slow jobs assigned evenly to each thread.

For ```.Q.fc```, the situation is reversed. This function splits a vector argument evenly along its length and assigns one slice to each secondary thread. 

```q
// NOW Unbalanced – first 4 fast arguments sent to first thread
h2"\\t .Q.fc[{f0 each x}] 10 10 10 10 1000 1000 1000 1000"
```
```q
// NOW Balanced distribution
h2"\\t .Q.fc[{f0 each x}] 10 1000 10 1000 10 1000 10 1000"
```
Alternating values in the list will result in balanced threads but contiguous blocks of large or small values will not.

## Memory Usage
Each secondary thread has its own heap, a minimum of 64MB.

```.Q.w[]``` only shows stats of main thread and not the summation of all the threads (total process memory). 

```q
h2".Q.w[]"
```

Since V2.7 2011.09.21, ```.Q.gc[]``` in the main thread collects garbage in the secondary threads too.

Let's remove some references to data on our process and run garbage collecter.
```q
h2"delete a0 from `."
h2"delete a1 from `."
h2".Q.gc[]"
```

Automatic garbage collection within each thread (triggered by a ```wsfull```, or hitting the artificial heap limit as specified with ```-w``` on the command line) is executed only for that particular thread, not across all threads.

Symbols are internalized from a single memory area common to all threads.


# What should I use parallel processing for?
## Query Optimization
The partitioned database structure in kdb+ is well suited to parallel processing. 

The ability to access sections of the database independently can be extended using secondary processes, with each secondary thread being assigned a date from the Where clause to process.

Lets load in a sample database.
```q
h2"\\l ./db/taq" /Please only run once to avoid subsequent problems- can you tell why?   
```

Get a list of all the dates in the trade table:
```q
h2"dates:distinct exec date from select date from trade"
h2"dates"
```
Get a list of 50 first syms in trade table:
```q
h2"symList:50#distinct exec sym from select sym from trade"
h2"symList"
```
Now run a select for IBM trades across all dates using ```each```:
```q
h2"\\t raze {select from trade where date = x, sym in symList} each dates"
```
And again using ```peach```:
```q
h2"\\t raze {select from trade where date = x, sym in symList} peach dates"
```

We can see how ```peach``` speeds up the query.

In practice, kdb+ handles multi-threaded HDB queries under the covers without the need for any additional functions. 

It will automatically distribute work across secondary processes and aggregate the results back to the main thread.


## Database Maintenance

Adding or modifying a column to a partitioned database can be slow when carried out on large tables. This operation can be improved using ```peach```.

Let's add a column to the quote table on the test database. This is done using the functions in the dbmaint.q script.
```q
// loading in the dbmaint.q functions
h2"\\l ../../Parallelization.Scripts/dbmaint.q"
```
We will be using the addcol function that comes with dbmaint.q script.
```
addcol:{[dbdir;table;colname;defaultvalue] / addcol[`:/data/taq;`trade;`noo;0h]
     if[not validcolname colname;'(`)sv colname,`invalid.colname];
     add1col[;colname;enum[dbdir;defaultvalue]]each allpaths[dbdir;table];}
```
Add a new column called newcol and fill it with nulls:
```q
h2"\\t addcol[hsym `$getenv[`AX_WORKSPACE],\"/db/taq\"; `quote; `newcol; 0N]"
```
Check column got added:
```q
system"ls ",getenv[`AX_WORKSPACE],"/db/taq/2020.01.31/quote"
```
Now lets delete column:
```q
h2"deletecol[hsym `$getenv[`AX_WORKSPACE],\"/db/taq\"; `quote; `newcol] "
```
Check column got deleted:
```q
system"ls ",getenv[`AX_WORKSPACE],"/db/taq/2020.01.31/quote"
```

Lets define a new function which is a modified version of addcol, which updates all partitions in parallel: addcolPeach.

```q
// Define addcolPeach using peach
h2"addcolPeach: {[dbdir;table;colname;defaultvalue]
    if[not validcolname colname;'(`)sv colname,`invalid.colname]; 
    add1col[;colname;enum[dbdir;defaultvalue]] peach allpaths[dbdir;table];}"
```
Let's try our modified form of addcol using peach :
```q
h2"\\t addcolPeach[hsym `$getenv[`AX_WORKSPACE],\"/db/taq\"; `quote; `newcol; 0N]"
```
This modified version takes less time than the standard ```addcol``` version.

If we had more cores at our disposal and could start more secondary threads this would show an even greater improvement.


## Writing/Reading Flat Files

Reading and writing data from disk, particularly in comma- or tab-delimited format, can be a time-consuming operation. 

We can use secondary processes to speed up this process, particularly when we need to write multiple files from one process e.g. saving down EOD data.

Let's take an extract from the trade table on the test database.
```q
// select 100 syms from first date
h2"symList100:100#distinct exec sym from select sym from trade"
h2"td: select from trade where date=first date, sym in symList100"
h2"select count i by date,sym from td"
```
Let's then cut the table into five slices:
```q
// get the size of each slice by dividing count table by 5 and flooring to nearest whole number 
h2"sliceSize:(floor (count td)%5)"
```
```q
h2"tds: sliceSize cut td"
h2"count each tds"         // can see we have created 5 tables of equal size
```
Now let's save each slice to disk with a separate filename:
```q
// each
h2"\\t {(hsym `$getenv[`AX_WORKSPACE],\"/tmpEach/testfile\",string 1+x) 0: csv 0: tds[x]} each til 5 "
```
```q
// peach
h2"\\t {(hsym `$getenv[`AX_WORKSPACE],\"/tmpPeach/testfile\",string 1+x) 0: csv 0: tds[x]} peach til 5 "
```
We can see ```peach``` takes less time to execute save down of the five files.

Let's check if the same is true for reading in the files.
```q
// define function to load in files from disk
h2"loadFiles:{1_/:(\"DSTFIC\";csv) 0: (hsym`$getenv[`AX_WORKSPACE],\"/tmpEach/testfile\",string 1+x)}"
```
```q
// each
h2"\\t eachFiles:raze loadFiles each til 5"
```
```q
// peach
h2"\\t peachFiles:raze loadFiles peach til 5"
```
Again, the parallel operation is faster at reading the data.

We can check that data is the same regardless of whether we use ```each``` or ```peach``` to write/load.
```q
h2"eachFiles~peachFiles"
```


### Further Reading:
* [Parallel Processing in kdb+](https://code.kx.com/q/basics/peach/)
* [Multi-threading in kdb+](https://code.kx.com/q/wp/multi-thread/#db-maintenance)
* [Distributed Each](https://code.kx.com/q4m3/A_Built-in_Functions/#a682-distributed-peach)
* [Balancing Slaves and Cores](https://code.kx.com/q4m3/14_Introduction_to_Kdb%2B/#1446-balancing-slaves-and-cores)
