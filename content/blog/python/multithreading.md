---
title: "Multithreading"
date: 2023-09-07T18:51:26+02:00
draft: true
---

* cpu bound problems still let other less intensive threads time on the cpu
  with sys.setswitchinterval (do an experiment and set the interval to 10
  seconds)
* what is the concurrent.futures library all about and how can
  ThreadPoolExecutors help with IO bound problems
* talk about the synchronization primitives in the threading module
* what the interpreter actually executes, dis.dis
* multitasking can be preemptive or cooperative

# Introduction

Few discussions regarding multithreading in Python can escape the infamous
GIL and the qualification that Python is lacking real multithreading support.
In this blog post we will explore, on a few examples whether those
qualifications have merit, and if they do, what conditions make them true.
Before we start exploring those questions, let's cover some basic concepts.

# Threads vs. Processes

The CPU has a very limited, specialized language it understands called the
instruction set. Whatever programming language you use, be it a statically
typed compiled language like C, or a dynamically typed interpreted language
like Python, the code you write has to be translated into a language the CPU
understands. Whether that translation happens by compiling and linking (C) or
by some intermediate agent like a virtual machine (Python), those
instructions are loaded from memory onto the CPU. Your program had just
become a process, started with at least one thread, the main thread.

## Processes

A process is a running instance of a program, having its own isolated memory
space. That means one process cannot interfere with the execution of another
process. It also means that interprocess communication is somewhat expensive
as data has to be serialized (deserialized) when sent (received) to (from) a
process. In Python, this usually involves pickling an object, which has its
own set of problems.

## Threads

A thread is a unit of execution (a collection of instructions) within a
process. Each process has at least one thread, called the main thread, but
many other threads can be spawned within a process. Each thread has its own
stack, but shares memory with other threads in the same process. Sharing of
memory makes programming with threads inherently difficult as more threads
can read/write from the same memory address, which can lead to race
conditions, and can leave your program in a corrupted state. This problem is
managed by locks and other synchronization primitives, but nothing forces a
programmer to do that. It's an approach that requires discipline on the part
of the programmer. As threads share memory, context switching is faster for
threads than processes.

## Multitasking and Scheduling

The CPU can only execute one set of instructions at a time on a single core.
If you have a four core CPU, only four sets of instructions can be executed
simultaneously. On the other hand, a modern operating system (OS) can run a
hundred processes, to what appears, simultaneously. The OS accomplishes that by
giving each process a slice of time on the CPU in a way that each process
makes progress. Modern CPUs are so fast that a user cannot notice that the
OS is actually switching continuously between processes, and that each
process spends a relatively small amount of time on the CPU.

To do that there are basically two strategies. One is called cooperative
multitasking, and the other preemptive multitasking. Cooperative
multitasking means that each process, well, has to cooperate with others,
yielding control to other processes, so that each can make progress. If a
process misbehaves and doesn't yield control to other processes, the whole
system can grind to a halt. In fact, that's what used to happen with older
operating systems. They only cure was a hard reset. Fortunately, modern
operating systems use preemptive multitasking, meaning that the OS gives a
limited slice of CPU time to a process. If a process misbehaves and doesn't
ceed control willingly, the OS removes it from the CPU and gives another
process a chance to run.

Cooperative multitasking is an excellent strategy but when employed on a
different level, i.e. when the cooperation becomes part of the language
implementation, like async programming in Python with coroutines or
goroutines in the Go programming language.

# Threads in Python

## The Global Interpreter Lock (GIL)

Let's dissect that infamous term, the GIL. First, it's global, meaning that it
applies to the whole Python program being executed. Second, it applies to
the interpreter, meaning the CPython interpreter that is actually executing
code. Finally, it's a lock, meaning it's tied to threading somehow. But what
does the interpreter actually interpret, or execute?

### Python Bytecode

The Python interpreter is actually a virtual machine that executes bytecode.
The job of the virtual machine is to turn that bytecode into machine code
the CPU understands. There is a neat module that allows disassembling Python
source code, or Python objects into bytecode. Suppose we have a simple
function that adds two arguments `a` and `b`

```python
def add(a, b):
    result = a + b
    return result
```

We can see the generated bytecode of that function in human-readable form by
using the `dis.dis`function

```shell
>>> import dis
>>> dis.dis(add)
  1           0 RESUME                   0

  2           2 LOAD_FAST                0 (a)
              4 LOAD_FAST                1 (b)
              6 BINARY_OP                0 (+)
             10 STORE_FAST               2 (result)

  3          12 LOAD_FAST                2 (result)
             14 RETURN_VALUE
```

The execution of bytecode by the Python virtual machine, or interpreter, is
single threaded, meaning only one thread within the Python process can
execute bytecode, while other threads can simply wait their turn. If you 
wanted to execute our `add` function in two threads, here would be one 
possible execution flow:

```text
thread | bytecode
1      | RESUME
1      | LOAD_FAST  (a)
1      | LOAD_FAST  (b)
1      | BINARY_OP  (+)
1      | STORE_FAST (result)
# context switch by the OS; OS decides when to switch the execution of threads
2      | RESUME
2      | LOAD_FAST  (a)
2      | LOAD_FAST  (b)
2      | BINARY_OP  (+)
# the OS switches threads again
1      | LOAD_FAST  (result)
1      | RETURN_VALUE
# thread 1 is complete
2      | STORE_FAST (result)
2      | LOAD_FAST  (result)
2      | RETURN_VALUE
# thread 2 is complete
```

If you're wondering why does Python have the GIL in the first place, it
makes the C implementation of the interpreter, reference counting, garbage
collection and many other aspects simpler.

### I/O Bound vs CPU Bound Problems

The above example illustrates that Python threads are useful for cooperative 
multitasking, which usually involves I/O bound problems. I/O bound problems 
have to do with a lot of waiting (waiting means that the CPU does a lot of 
cycles without executing any of our code), like waiting for a server 
response using `requests.get` or `httpx.get`, waiting for a socket, or 
simply waiting for `time.sleep(1)` to elapse.

CPU bound problems, on the other hand, keep the CPU busy, and such threads 
cooperate poorly with others as each thread  wants to keep the CPU as busy as 
possible. An example of CPU bound problems would be anything computationally 
heavy, like multiplying matrices or calculating prime numbers.

If your problem is CPU bound, multiprocessing would be a better solution. In 
fact, trying to solve a CPU bound problem with threads in Python would be 
slower than a single threaded solution because of the context switch 
overhead. If, on the other hand, your threads do nothing most of the time, 
meaning they are simply waiting for some resource, multithreading is a good 
approach.

Now that we have some theory under our belt, let's move on to some examples.

## Producer Consumer Problem

A common problem solved by threads involves one thread producing some data,
and another consuming or processing that data. Here is an example

```python
import queue
import threading
import time

q = queue.Queue()
poison_pill = object()


def producer():
    for i in range(10):
        print('producing', i)
        q.put(i)
        time.sleep(1)
    print('producer done')
    q.put(poison_pill)


def consumer():
    while True:
        item = q.get()
        if item is poison_pill:
            break
        print('consuming', item)
    print('consumer done')


t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)

t1.start()
t2.start()
```

There are a couple of things to note here. First, the producer thread sleeps 
a lot, and the consumer thread does some lightweight processing, so our 
problem is I/O bound, which means threads are a good approach. Second, since 
two threads are reading and writing from/to the same data structure, those 
operations need to be thread-safe. The `queue` module is designed with 
exactly that in mind. However, be aware that not all methods on the
`queue. Queue` object are thread-safe. Second, the `q.get()` blocks if there 
are no items in the queue. This can leave the consumer thread hanging in 
that blocking call if the producer thread is done. To avoid that, it is 
common to send a so-called poison pill item to signal to the consumer thread 
that it should stop processing data.

## Token Bucket

Suppose you have a web server, or some other resource the usage of which you
want to limit. One strategy you might want to employ is a token bucket. A
token bucket holds a limited number of tokens that are being refilled at
regular intervals. Each access to a limited resource consumes a token. If
all tokens have been consumed, the resource is unavailable until refilled
with tokens again. Here is one possible implementation of a token bucket.

```python
import logging
import queue
import threading
import time

logging.basicConfig(
    format='[%(levelname)s %(asctime)s %(threadName)s]: %(message)s'
)
logging.getLogger().setLevel(logging.INFO)

_token = object()


class TokenBucket:
    def __init__(
        self,
        refill_period: float,
        refill_tokens: int,
        max_tokens: int,
    ):
        self._refill_period = refill_period
        self._refill_tokens = refill_tokens
        self._bucket = queue.Queue(maxsize=max_tokens)
        self._refill_thread = threading.Thread(
            target=self._refill_target, name='RefillThread'
        )
        self._stop_event = threading.Event()

    def start(self):
        logging.info('Starting refill thread.')
        self._refill_thread.start()

    def stop(self):
        logging.info('Setting stop event to true.')
        self._stop_event.set()

    def request(self):
        try:
            self._bucket.get(block=False)
        except queue.Empty:
            return 'denied'
        else:
            return 'success'

    def _refill_target(self):
        while True:
            if self._stop_event.is_set():
                logging.info('Stopping refill thread.')
                break
            logging.info(f'Refilling bucket with {self._refill_tokens} tokens.')
            for _ in range(self._refill_tokens):
                try:
                    self._bucket.put(_token)
                except queue.Full:
                    break
            time.sleep(self._refill_period)
        logging.info('Refill thread completed.')
```

You can test out the above example in the Python console like this

```shell
>>> tb = TokenBucket(10, 2, 5)
>>> tb.start()
```

Now call `tb.request()` repeatedly and observe what happens. If you try to 
exit the Python interpreter while `TokenBucket` is still running, you might
be surprised that the interpreter hangs. That's because the `refill` thread 
was not stopped. If you do call `tb.stop()`, the `refill` thread will exit 
cleanly, but only because we implemented a mechanism for clean termination. 

In other words, if you haven't set up a mechanism so that a thread can exit 
cleanly, there is nothing you can do to stop it except terminating the whole 
Python process.

Another somewhat surprising aspect of programming with threads is the impact 
of unhandled exceptions one thread has on another. The surprise is that 
there is no impact. If one thread fails with an unhandled exception, other 
threads will proceed as if nothing happened. Although threads share memory, 
each has its own stack and isn't interested in what is happening with 
other threads.

Now that we have seen a concrete example involving threads, let's explore 
how Python behaves under various types of load.

# Impact of the GIL

I mentioned earlier that there are two types of problems in Python regarding 
CPU load; I/O bound and CPU bound. Here is an example of such functions

```python
import time

def io_bound(name: str, iterations: int, sleep: float):
    for i in range(1, iterations + 1):
        print(f'[io-bound-{name}] {i}/{iterations}')
        time.sleep(sleep)


def cpu_bound(name: str, iterations: int):
    delta = round(iterations / 100)
    for i in range(1, iterations + 1):
        q, r = divmod(i, delta)
        if r == 0:
            print(f'[cpu-bound-{name}] {q}/100')
```

First, let's take a look at the performance impact of running `io_bound` in 
multiple threads. For our baseline, we will run the `io_bound` function in 
the main thread and measure its performance with `time.perf_counter`.

```python
t1 = time.perf_counter()
io_bound('1', 20, 1)
t2 = time.perf_counter()

print(f'elapsed: {t2 - t1}')
```

On my machine, this gives `20.016027782010497` seconds. Nothing unexpected 
here. Let's give it a try with several threads running the `io_bound` function.

```python
num_threads = 20
io_bound_threads = [
    threading.Thread(target=io_bound, args=(str(i), 20, 1))
    for i in range(1, num_threads + 1)
]

t1 = time.perf_counter()
for thread in io_bound_threads:
    thread.start()
for thread in io_bound_threads:
    thread.join()
t2 = time.perf_counter()

print(f'elapsed for {num_threads} threads: {t2 - t1}')
```

The `thread.join` statement means that the main thread, i.e. the thread 
which starts all other threads will wait until each thread completes before 
evaluating the `t2` time. For the `io_bound`function running in 20 threads, 
the total elapsed time was `20.035348607998458`. When threads cooperate, as 
they do in this example, multithreading is a great approach for solving 
concurrent problems. Here, each thread does a little work, and releases the 
GIL (`time.sleep` releases the GIL, as do many other system calls) so that 
some other thread can proceed.

We will examine three cases
* `io_bound` function executed in multiple threads
* `cpu_bound` functions executed in multiple threads
* `io_bound` and `cpu_bound` function executed, each in one thread