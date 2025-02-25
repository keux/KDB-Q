# Practical Guidance - Parallel Processing

# Global Variables
Code executed in a secondary thread cannot update global variables – i.e., a function can only update local variables.
```q
// setting a on secondary thread
h2"{a::x} peach enlist 0"
```
This works, as single-item list shortcuts to execute on the main kdb+ thread.

But this:
```q
h2"{a::x} peach 0 1"
```
fails and signals ```noupdate``` as it is executed from within secondary threads.

We can actually use this distinction to check if we are in a main or secondary thread by exploting this distinction that we can't set global vars in secondary thread.

Using error trap we try to update global variable ```a``` :
* if it passes then return `main
* if it fails then return `thread
```q
h2"{@[value;\"{a::1;`main}[]\";`thread]} each 1 2"
```
```q
h2"{@[value;\"{a::1;`main}[]\";`thread]} peach 1 2"
```



# [Multithreaded Primitives](https://code.kx.com/q/kb/mt-primitives/)

kdb+ 4.0 introduced implicit, within-primitive parallelism which complement existing explicit parallel computation facilities (```peach```).

This module is created using v3.6 so won't be using implicit parallelism - but download 4.0 locally yourself to see the difference! 
```q
.z.K
```

The following primitives now use multiple threads where appropriate:
|           |                                                                                                                                                           |
|-----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| atomics      | abs acos and asin atan ceiling cos div exp floor log mod neg not null or reciprocal signum sin sqrt tan within xbar xexp xlog + - * % & \| < > = >= <= <> |
| aggregate    | all any avg cor cov dev max min scov sdev sum svar var wavg                                                                                               |
| lookups      | ?(Find) aj asof bin binr ij in lj uj                                                                                                                      |
| index        | index@(Apply At) select .. where delete                                                                                                                   |
| misc         | $(Cast) #(Take) _(Drop/Cut) ,(Join) deltas differ distinct next prev sublist til where xprev select ... by**                                              |
                                                                                                                                                           
 Multithreaded primitives execute in the same secondary threads as ```peach```, and similar limitations apply. 

To keep overhead in check, the number of execution threads is limited by the minimum amount of data processed per thread – at the moment it is in the order of 10^5 vector items, depending on the primitive.

### Advantages of implicit primitives:
* No overhead of splitting and joining large vectors. For simple functions, direct execution can be much faster than ```.Q.fc```
```
C:\Users\mwoods>q -s 24
KDB+ 4.0 2020.07.15 Copyright (C) 1993-2020 Kx Systems
w64/ 8(8)core 7998MB mwoods lptp1291 172.24.0.1 EXPIRE 2022.02.18 mwoods@kx.com #62736

q)system"s 24";a:100000000?100
q)\t a*a
1430
q)\t .Q.fc[{x*x};a]
5743
```

**However, one needs vectors large enough to take advantage. Nested structures and matrices still need hand-crafted peach. Well-optimized code already making use of peach is unlikely to benefit!**

* Operating on one vector at a time can avoid inefficient scheduling of large, uneven chunks of work
```
C:\Users\mwoods>q -s 3
KDB+ 4.0 2020.07.15 Copyright (C) 1993-2020 Kx Systems
w64/ 8(8)core 7998MB mwoods lptp1291 172.24.0.1 EXPIRE 2022.02.18 mwoods@kx.com #62736

q)system"s 3";n:100000000;t:([]n?0f;n?0x00;n?0x00);
q)\t sum t            / within-column parallelism
160
q)\t sum peach flip t / column-by-column parallelism ..
202
q)\s 0
q)/ .. takes just as much time as the largest unit of work,
q)\t sum t`x          / .. i.e. widest column
202
```
# [Distributed Each](https://code.kx.com/q/basics/peach/#processes-distributed-each)

Since V3.1, peach can use multiple processes instead of threads, configured through the startup command-line option -s with a negative integer, e.g. -s -4.

This is called Distibuted Each, may previously be known as Negative Slave Count.

For ```q -s N```:
| N  | Application                   |
|----|-------------------------------|
| >0 | number of secondary threads   |
| <0 | process with handles in .z.pd |


Starting q with –s –N indicates:

* In addition to the main q process, you will instantiate n workers, where a worker is an independent q process with an open port
* You will connect the master q process to each worker
* You will provide the master process a unique’d list of open handles to the workers

The list of handles to the workers is obtained from the system variable ```.z.pd``` upon each invocation of ```peach```. You can set ```.z.pd``` either to an integer list of open handles with the `u# attribute applied, or a function that returns the same.

*Note you must apply the `u# attribute or you will get the error:*
```
'.z.pd - expected unique vector of int handles
```

In contrast to the situation with slave threads, where the threads are created for you automatically, with distributed peach it is your responsibility to instantiate the separate worker processes and make them available to the main q process. 

A real life use case of distributed each would be multiprocess HDBs.


# Observations 
We collect here observations on execution within a slave. See the KX site for more details.

* If the list has a single item, peach application occurs in the main q thread only.

* If the function applied with peach performs grouping on a symbol list – e.g., a select that groups on a ticker symbol – this will be significantly slower in the slave because the optimized algorithm used in the main thread is not available in the slave.

* A socket can be used from the main thread only. As there is no locking around a socket descriptor, a communication handle shared between threads would result in garbage due to message interleaving.

* Each slave thread has its own heap, a minimum of 64MB. Executing .Q.gc[] in the main thread triggers garbage collection in the slave threads too. Automatic garbage collection within each thread is only executed for that particular thread, not across all threads.

* Symbols are internalized in a single memory area accessible to all threads.

# Timer with secondary threads
If you use a timer on the main thread or other tricks to update globals that are read by slave threads, you are cruising for a bruising. While these updates are locked, there is no notion of a transaction with attendant rollback. It is unlikely that your function executing on the timer is atomic at the hardware level. Should it fail mid-flight, you should expect the workspace would be in an inconsistent state with no built-in recovery mechanism.


## Further Reading:
* [Parallel Processing in kdb+](https://code.kx.com/q/basics/peach/)
* [Multi-threading in kdb+](https://code.kx.com/q/wp/multi-thread/#db-maintenance)
* [Distributed Each](https://code.kx.com/q4m3/A_Built-in_Functions/#a682-distributed-peach)
* [Balancing Slaves and Cores](https://code.kx.com/q4m3/14_Introduction_to_Kdb%2B/#1446-balancing-slaves-and-cores)