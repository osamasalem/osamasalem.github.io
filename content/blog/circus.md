+++
title = "Circus: Yet Another Programming Take in the Age of Multi-Core Processing"
date = 2019-11-27
slug = "circus-prgramming-in-multicore-era"
+++

### Motivation
Since mid of last century. the programming languages works in the same way like they are first invented, and as the technology moves monotonously and continuously, and the hardware hitting the hard limit of physics and switching to multi-core processing architecture , the need to find another view of point of how can we can solve our increasing needs of computing to another dimension,

### What is Circus ?
Circus is a actor model runtime environment with embedded programming language, that break the logic in parallel elementary atomic actors that receives multiple inputs and produces output as workloads messages sent to each other to model the needed logic.

For instance something like `5 * 4 + 3 * 6`, we can execute the multiplications in parallel and add their results after,

Those operations is held in binary machine code created by low level languages as dynamic linked modules at runtime and loaded lazy fashion at runtime once needed

Atomic here means that those actors represent very miniscule operations ( you can think of it like arithmetics and boolean operations) and holds guarded state, atomic memory portions, or holds no-state at all, those actors may be either input-only, input-output, or output only,

So the messages are queued in multiple of very performant network-like LIFO ring-buffered queues and consumed by the different buffers on actors execution (dequeue), after the actors process the data, it enqueues the output to the buffers again to be processed by other actors and so on,

you can think Circus is a special dialect of Prolog, but it is used for broader general purpose programming addressed to solve software problems and works with modern CPUs/GPUs in mind, with simpler concept and syntax

Despite its general purpose nature. I see it is most fitting some domains like data and media streaming and network processing and so on

### Hello World

Here is hello world example
```rs
fn main():(int ret) {
    puts s("Hello, World!!");
    ret = signal_int(s.done, 0);
}
```


### From where does this idea come from?

The first idea was about drawing programs rather than writing them, so how can we graphically represent a program by a sketch like block diagram

then when I worked in networking and multithreading, this idea has grown with multiple details in mind, as described below

### Why Circus ?
This for a number of reasons
1. the current traditional apps that relies on single threads, does not utilize the full power of the newer processor
2. The actors and messaging paradigm is one of the safest and efficient model for parallelization programming (consider Channels in Erlang, Go and Rust).
3. Replacing stacks with queues grants some versatility and flexibility rather than stack in managing memory and preserve internal cache lookups


### Circus anatomy
the Circus program is a graph consisted of different micro actors, Each actors -and also called components- is an abstract interface with single function for execution, and each component has its own arguments and return values -lets call them input pins and output pins respectively' -

Circus graph is instantiated from multiple actor instances, those actors are connected by their respective pins to each other by `connections`, this connections is a logical entity about the source of the data processed by the actor and destination of the data sent by the actor.

So in contrast of traditional languages which is a set of multiple instructions, In Circus it is like a web or graph where instructions are connected together

In runtime execution , the runtime creates multiple threads `jugglers` with the number of cores in the cpu , and creates a number of ring buffer workload queues equal to 1.5 number of CPU cores, each queue has internal size = 1MB RAM which are consumed with enqueued tasks and freed by work tasks done , those values sure are customized based on needs but those are the defaults,

And then those threads search for workloads in those queues, fetch them and direct them to the actors executors, and execute the corresponding underlying functions respective, and when the thread finish the actor work, it looks up for another workload task and so on

### The Ring master
The `Ring master` is a special low priority main thread responsible of managing the runtime, so here is some of his duties

* ends the program when there is no more tasks in all queues within set timeout (say 5secs)
* Adaptively, so he can have the authority to increase the number of queues or threads based on the state of those jugglers and queues
* He can dismiss one of the jugglers when it is stuck with a task within set timeout, he can reclaim the task and enqueue it somewhere else, and restart the thread again

### Using Data-tags
Circus Runtime is not aware of data types and their sizes at runtime, it just keep track of what so called data tags defined in the script language, this data tags are checked strictly by the runtime when connecting two actor's pins together, it makes sure that the output pin of the source actor has the same data tag of the destination input pin of receiving actor.

I would like to reassure that those tags are just names with empty meaning for the runtime, so the runtime is not aware of any memory size and alignment of the date sent to/received from the actors, and its actor responsibility to make sure that the data they deal with are compatible

Sure there are `int`,`float`,`char`,`bool` reserved tags , but the runtime is checking those tags but agnostic about their meaning

and also there are literals that runtime can translate it to data types, but this is what can it do

there also the void type connections; which is type-less signals, which does not hold data, and does not has hash (as discussed soon)
### Types of actors

There are two types of actors
* **Binary actor**: which is the function imported from the binary dynamic-linked library and represented by machine code,
* **Logical actor**: it is something like functions in traditional languages,that contains multiple binary and logical actors graph packaged as a black box
### Logical actors

we can define the actors using this form
```rs
fn foo(int a, int b):(int ret) {
    int c = a + b;
    string str = (string)c;
    puts(str);
    ret=1234;
}
```
- a+b is executed and assigned to c, and 1234 is sent through ret pin in the same time supposedly .
- then a + b result is assigned to c;
- then c is converted to string;
- then str is printed

Notice that the ret assignment is not dependent on earlier instructions

### No recursions, But repetitions
So as the structural nature of the runtime, there is no recursion, as the definition of the logical actor is based on the definition of the child actors, so it is impossible to define recursive actors

But repetitions are possible, consider the following
```rs
x = 0
x = x + 1
```
This will do infinite loop

so x here is a cache actor the takes value and sending it again (not typical a variable) so here x value takes 0 and then send it add which will add it to 1 and send it back to x which will send it to add and so on to infinity, and this program will never end,

### The data structure
The data is generally copied into the workloads, so it is not a good practice to hold pointers or references unless the data is so huge, and in this case the responsibility of the actors to keep the shared data safe some copy-on-write ref-counted well-guarded memory area, or this shared state can be held in well-guarded actors accessible from different other actors

another idea is that the ring master can dedicate a special Agent for those data if needed

### Don't recompute the wheel
Consider we have a video game, where the player does not move a mouse, or register any key strokes, it is inefficient to redraw the same scene multiple times , which consumes energy and processing power to do the same thing

In Circus runtime the data propagates by change, so actors should not process the same values repetitively in consecutive cycles,

To prevent to initiate long chain reaction for data that does not change, and save processing of data that is already computed before.

Each piece of data should has some sort of 64-bit hash value to prevent re-processing the same data twice in the same pin. so output pins have hash value of the last data sent, and in each occasion that hashes are equal the data should be held and not posted, unless otherwise specified,

consider the modified version of last example ( multiplication instead of addition)
```rs
x = 0
x = x * 1
```
Here the program will end because the x mul operation will have the same hash from the last cycle

Primitive data has hash of itself ,so no need for hashing here

### Syncing using sequence number ?
the runtime may have the capability to identify expired values that approaches the input pin using `global sequence numbers` , every value post in output pin dispatch an integer representing the sequence of this value .. so the runtime can ignore the older value if the newer value received first
this mechanism is similar to what is widely used in networking to eliminate any out-of-order packets in favor for newer ones

### The dual nature of actors
the actors in Circus has dual nature, to be function with arguments or object with properties
for instance lets consider this example


```rs
// we have a function like this
fn foo(string s, float f):()

// we can say this
foo("abc", 1.2);

// --OR--
foo f;
f.s = "abc";
f.f = 1.2

// or even
foo f("abc", 1.2);

```

### Joining and Forking
there is two types of necessary actors
* **Joins**: which has two inputs one output of the same data tag, whenever an updated value is received by one of the two input pins ,it will be directed to the output pin
* **Forks**: which has one input two outputs of the same data tag, whenever a new value is approaching is replicated in both output pins

## Horizons
- Hardware : May be it is a good opportunity to reconsider the hardware architecture which can adopt this model to expand and grant more performance for end users
- multi-tenant deployment synchronization: Passing tasks between runtimes in different server machines can grant good opportunities for scaling, availability, fault tolerance, and distributed systems
- Utilize Modern control theory to manage runtime resources: may be it is fortunate for runtime to adapt the number of jugglers/queues based on the load introduce to the runtime





