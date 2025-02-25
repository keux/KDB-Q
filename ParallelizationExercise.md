# Parallel Processing Exercises

Before we start the Quiz, let's launch a remote q process that has 2 secondary threads: 
```q
system["q  -p 5097 -s 2 >/dev/null 2>&1 &"];
```

### Exercise 1
Open a connection to process defined above (port 5097) and save as varaible ```hs```:
```q
// your code here
```


### Exercise 2
Check the number of cores available on our machine:
```q
// your code here
```


### Exercise 3
Check the number of secondary threads using handle ```hs```:
```q
// your code here
```


### Exercise 5
Write some code to generate a list of 50 random floats under 1.0. They should all be positive.
```q
// your code here
```

### Exercise 6
Raise ```e``` to a power which is the result from Exercise 5.

*Hint check out [exp](https://code.kx.com/q/ref/exp/)*
```q
// your code here
```


### Exercise 7
Create a function ```sumExp``` that:
* raises e to a power with is a list of ```x``` floats
* sum this result
* define this function on process ```hs```
```q
// your code here
```


### Exercise 8
Run this function for 50,000,000 floats using each saving result to variable ```fEach```, and time the execution:
```q
// your code here
```


### Exercise 9
Run this function for 50,000,000 floats using peach saving result to variable ```fPeach```, and time the execution:
```q
// your code here
```


### Exercise 10
Given the results from Exercises 8 & 9 decide whether you should use each or peach for the most performant code:
```q
// your code here
```


### Exercise 11
Create a new function ```xexpFunc``` that:
* raises x to the power of 1.9
```q
// your code here
```


### Exercise 12
Run this function ```xexpFunc``` using ```peach``` and passing in a vector ```vec:til 1000000```. Save the result to variable ```fPeach2``` and time execution:
```q
// your code here
```


### Exercise 13
Run function ```xexpFunc``` using .Q.fc instead. Save the result to varaible ```qfc2``` and time execution:
```q
// your code here
```

### Exercise 14
Given the results from Exercises 12 & 13 decide whether you should use peach or .Q.fc for the most performant code:
```q
// your code here
```
