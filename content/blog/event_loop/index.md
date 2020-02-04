---
title: Event Loop in Python
date: "2020-02-04"
description: An attempt to implement Event Loop in Python
---

In the [last post], we implemented futures in python. We used threads to complete tasks that weren’t trivial computations (i.e, operations that took *more* time, like reading a file, waiting for a given amount of time etc.). In this post we shall see how to implement a *runtime* that can handle these time-taking operations in a *single thread*, with the help of operating system (*this means that we will be offloading the tasks to the operating system*).

> NOTE: I will be using Linux here. The code may not run on other platforms. But most of the platforms have some kind of an analogue to all the system calls used here

The implementation is inspired from *asynchronous runtimes* like *NodeJS* and *TokioRS*. The concept shall remain the same, albeit our implementation will be simpler, providing support only for *delay* and *reading from sockets*. 
> I wanted to make this still simpler by reading from normal files, but the linux syscall used in implementation, **epoll** doesn’t support the regular files, so I resorted to using sockets.

Even though the runtimes like *NodeJS* and *TokioRS* are extremely complex and can seem like magic, the principle behind them is pretty simple, once you dig into their source. They contain a loop, called the ***Event Loop***. The *event loop* is a ***semi-infinite loop*** (This loop ends when there is no longer need for it, i.e, there are no more tasks to be scheduled onto the thread and there are no tasks waiting for events to occur). [More Info](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
The NodeJS event loop works in *stages*. Single iteration of the loop is called a ***tick***. So every tick contains the following stages:

* **Timers**: this phase executes callbacks scheduled by setTimeout() and setInterval().
pending callbacks: executes I/O callbacks deferred to the next loop iteration.
* **Idle, prepare**: only used internally.
* **Poll**: retrieve new I/O events; execute I/O related callbacks (almost all with the exception of close callbacks, the ones scheduled by timers, and setImmediate()); node will block here when appropriate.
* **Check**: setImmediate() callbacks are invoked here.
* **Close callbacks**: some close callbacks, e.g. socket.on('close', ...).

Again, more information on these stages can be obtained from the [official Node docs](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/).

The runtime/event loop that we will implement will take help from the *futures* that we have already implemented. Our event loop will contain only two stages: 
* **Timers** 
* **Poll**

Node uses only a *single thread* to run both the JS code and also to monitor the files Node ([or libuv](https://libuv.org/)) uses *syscalls* provided by the operating system to achieve this. The operating system has multiple threads, which can be used to keep track of the files and notify the user space program about it.

We will run our event loop on a **separate thread** and *communicate with it from the main thread to add events to watch for and also to get back the results of the asynchronous work on completion*. Like Node, we will also be leveraging on the operating system provided file-descriptor-monitoring syscalls, in our case, it will be ***epoll*** (*linux-specific*). 
> Our code will run only on linux platforms. Windows and MacOS provide IOCP and KQueue for similar purposes. They vary in the details of how they work.

So, our implementation is ***tailored to our needs of being simple and using futures***.

First, let me define a few terms I will be using. ***Tasks*** refer to the top level things to be achieved. Any two tasks are *independent of each other*. The tasks are made up of several ***stages***. Stages are the subproblems to be solved in a sequential order, in order to complete the entire task. The stages are just *futures* in our implementation. The tasks run *concurrently*. There is almost no inter-dependence between two tasks. The stages of the same task are related by the *Happens-Before relationship*, where one stage/future has to be completed before the next one starts. Also the former stage has to passes its result to the latter (*composition*).

> I won’t be getting into the details of the code in this post. However, I will show how the above mentioned ideas can be manifested into code. 

The information required by the runtime is encoded in the class called ***Runtime***. 

```python
class RunTime():
    def __init__(self):
        self.timers = OrderedList()   # Ordered list of time objects
        self.poller = select.epoll()
        self.file_map = {} # every future will have different fileno, even though same file is 'opened'
        self.pending = 1
```
First, we need to handle timers. Our tasks can include stages that want us to wait for a certain time before moving on to the next one. In order to represent this requirement, we shall define ***Delay*** future. It will be the same as the ***Delay*** future defined in the previous post. But instead of making use of another thread for waiting, *it will ask our event loop to complete the future after the waiting time*. 
```python
def Delay(value, ms, runtime):
    future = Future()
    TimePoller(ms, value, future, runtime)  # completes (either pass or fail) after some time with a Result
    return future
    
def TimePoller(resolve_time, resolve_value, future, runtime):
    t = LoopTime(resolve_time, resolve_value, future)
    runtime.register_timers(t)
```
***Looptime*** object is used to *map* the timer to the future that gets resolved when the time has passed. It also contains the value with which the future should be resolved after the specified time. But we need an *ordered list*, with which we can efficiently perform *insert, search and delete* operations on the set timers. We can use the *bisect* python package to maintain an ordered list that is ordered by the time when the corresponding future has to be resolved.
The ***register_timers*** method on the ***runtime*** object, as the name says, is used to register the timer with the ***runtime***.

```python
 def FileRead(filename, runtime):
    future = Future()
    FilePoller(filename,future, runtime)
    return future
    
def FilePoller(port, future, runtime):
    try:
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        sock.bind(('0.0.0.0', port))
        sock.listen(1)
        sock.setblocking(0)
    except Exception as e:
        future.complete(Result(error=e))
    else:
        f = LoopFile(sock, future)
        runtime.register_files(f)

```
Next, we need to handle file reads, or the *socket* reads to be specific ( as mentioned before, epoll doesn’t support normal files). Same as in timers, we use a modified ***FileRead*** future from the last post, which instead of spawning a new thread to read the file, *registers a ***LoopFile*** object with the event loop, which then sends back the file contents after the read operation is finished*. ***LoopFile***, just like ***LoopTime***, is used to contain information about a file (or socket), like, the *future associated with it, contents read etc*. It works together with the ***file_map*** of the runtime object to achieve this. The file or *socket is read in a non-blocking way using the epoll*, in the ***edge-triggered mode*** .
> Remember to open the socket in non-blocking mode

Runtime also contains a ***poller***, which is an instance of *epoll*. This is used to monitor the files and also complete the corresponding futures, when the file is read or the time has passed. 
Runtime also has a method called ***loop***, which is **our actual event loop**. It’s a *semi-infinite loop* which terminates when the number of pending operations goes to zero. We run the method in a **separate thread**. 
```python
def loop(self):
    while self.pending > 0:
        # timers
        a = self.timers.remove_completed_timers()
        for i in a:
            i.future.complete(Result(value=i.resolve_value))
        # poll
        time_limit = self.timers.get_min_ms() / 1000
        self.poll(time_limit)
```
In every *tick* (or iteration) of the loop, the *expired timers are removed* and the *corresponding futures are completed*. Then the *files(sockets) are polled*, and if possible ***completed*** using the poll method, which uses the epoll and edge-triggered, non-blocking file descriptors to achieve the task. 
In order to account for the possible errors, our runtime completes the futures with ***Result*** objects specified in the previous post.  In short, the ***Result*** objects are *monads* similar to ***Result*** enum in Rust or ***Either*** in Haskell. The ***Future*** objects know how to handle a ***Result*** object. 

```python
def poll(self, time_limit):
    events = self.poller.poll(timeout=time_limit)
    for fileno, _ in events:
        if not self.file_map[fileno].connection:
            connection, _ = self.file_map[fileno].file.accept()
            connection.setblocking(0)
            self.poller.register(connection.fileno(), select.EPOLLET | select.EPOLLIN)
            self.file_map[connection.fileno()] = self.file_map[fileno]
            self.file_map[connection.fileno()].file = connection
            self.file_map[connection.fileno()].connection = True
            del self.file_map[fileno]
            self.poller.unregister(fileno)
        else:
            try:
                while True:
                    l = self.file_map[fileno].file.recv(16)
                    if not l:
                        self.file_map[fileno].future.complete(Result(value=self file_map[fileno].resolve_value))
                        self.file_map[fileno].file.close()
                        del self.file_map[fileno]
                        self.poller.unregister(fileno)
                        break
                    self.file_map[fileno].resolve_value += str(l)
                
            except Exception as e:
                if e.args[0] == errno.EWOULDBLOCK:
                    pass
                else:
                    self.file_map[fileno].future.complete(Result(error=e))
                    self.file_map[fileno].file.close()
                    del self.file_map[fileno]
                    self.poller.unregister(fileno)
```
The ***poll*** method reads the contents from the socket after a connection is formed. The contents are read in a *non-blocking* fashion. The poller works with the sockets in the ***edge-triggered mode***. So it's the responsibility of the user (of poller) to completely read the socket, before moving on, else data can be lost.
> The first message on the listening socket is a connection request. Form the connection and then start reading from the connection. Suitably, state has to be maintained.

So, as you can see, the concepts behind the event loop and asynchronous runtimes is pretty simple. What makes them complex is the presence of a wide array of moving parts, which have to work asynchronously. 
[TokioRS](https://tokio.rs/) and [async-std](https://async.rs/) are two such asynchronous runtimes that I am interested in.  
