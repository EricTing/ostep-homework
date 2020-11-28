
# Overview

This homework lets you play around with a number of ways to implement
a small, deadlock-free vector object in C. The vector object is quite
limited (e.g., it only has `add()` and `init()` functions) but is just
used to illustrate different approaches to avoiding deadlock.

Some files that you should pay attention to are as follows. They, in
particular, are used by all the variants in this homework.
- `common_threads.h`: The usual wrappers around many different pthread (and other) library calls, so as to ensure they are not failing silently
- `vector-header.h`: A simple header for the vector routines, mostly defining a fixed vector size and then a struct that is used per vector (vector_t)
- `main-header.h`: A number of global variables common to each different program
- `main-common.c`: Contains the main() routine (with arg parsing) that initializes two vectors, starts some threads to access them (via a worker() routine), and then waits for the many vector_add()'s to complete

The variants of this homework are found in the following files. Each takes a
different approach to dealing with concurrency inside a "vector addition"
routine called `vector_add()`; examine the code in these files to get a sense of
what is going on. They all use the files above to make a complete runnable
program. 

The relevant files:
- `vector-deadlock.c`: This version blithely grabs the locks in a particular order (dst then src). By doing so, it creates an "invitation to deadlock", as one thread might call `vector_add(v1, v2)` while another concurrently calls `vector_add(v2, v1)`.
- `vector-global-order.c`: This version of `vector_add()` grabs the locks in a total order, based on address of the vector. 
- `vector-try-wait.c`: This version of `vector_add()` uses `pthread_mutex_trylock()` to attempt to grab locks; when the try fails, the code releases any locks it may hold and goes back to the top and tries it all over again.
- `vector-avoid-hold-and-wait.c`: This version ensures it can't get stuck in a hold and wait pattern by using a single lock around lock acquisition.
- `vector-nolock.c`: This version doesn't even use locks; rather, it uses an atomic fetch-and-add to implement the `vector_add()` routine. Its semantics (as a result) are slightly different.

Type `make` (and read the `Makefile`) to build each of five executables. 

```sh
prompt> make
```

Then you can run a program by simply typing its name:

```sh
prompt> ./vector-deadlock
```

Each program takes the same set of arguments (see main-common.c for details):
- `-d`: This flag turns on the ability for threads to deadlock. When you pass `-d` to the program, every other thread calls `vector_add()` with the vectors in a different order, e.g., with two threads, and `-d` enabled, Thread 0 calls `vector_add(v1, v2)` and Thread 1 calls `vector_add(v2, v1)`
- `-p`: This flag gives each thread a different set of vectors to call add upon, instead of just two vectors. Use this to see how things perform when there isn't contention for the same set of vectors.
- `-n num_threads`: Creates some number of threads; you need more than one to deadlock.
- `-l loops`: How many times should each thread call add?
- `-v`: Verbose flag: prints out a little more about what is going on.
- `-t`: Turns on timing and shows how long everything took.


# Questions and Solutions

## basic usage
```sh
➜  threads-bugs git:(master) ✗ ./vector-deadlock -n 2 -l 1 -v 
->add(0, 1)
<-add(0, 1)
              ->add(0, 1)
              <-add(0, 1)

threads: 2 
loop: 1 
vector_add_order: 0 (adding v1 to v0)
initial values of v0: {0, 0, 0 ...}
initial values of v1: {1, 1, 1 ...}
final values of v0: {2, 2, 2 ...}
final values of v1: {1, 1, 1 ...}
```

## enable deadlock
```sh
➜  threads-bugs git:(master) ✗ ./vector-deadlock -n 2 -l 1 -v -d
->add(0, 1)
<-add(0, 1)
              ->add(1, 0)
              <-add(1, 0)

vector add order for T0: adding v1 to v0
vector add order for T1: adding v0 to v1
possibility to deadlock due to circular wait
```

## try-wait v.s. global-order v.s. avoid-hold-and-wait
```sh
➜  threads-bugs git:(master) ✗ ./vector-try-wait -n 2 -l 10000000  -d  -t
Retries: 101664246
Time: 17.46 seconds
➜  threads-bugs git:(master) ✗ ./vector-global-order -n 2 -l 10000000  -d  -t 
Time: 1.78 seconds
➜  threads-bugs git:(master) ✗ ./vector-avoid-hold-and-wait -n 2 -l 10000000  -d  -t 
Time: 2.50 seconds
➜  threads-bugs git:(master) ✗ ./vector-nolock -n 2 -l 10000000  -d  -t 
Time: 35.64 seconds
```
* try and wait wastes time on retrying;
* global-order is the smartest strategy with high concurrency;
* avoid-hold-and-wait introduces a global lock and all threads must be acquired early on (at once) instead of when they are truly needed.
* why `vector-nolock` is the slowest?
