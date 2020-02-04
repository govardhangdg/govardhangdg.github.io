---
title: Futures in Python
date: "2020-02-03"
description: An attempt to implement Futures in Python
---

In the last post, we saw the importance of composition in programming. Futures are a way to effectively introduce composition in concurrent programming. In this post we will see how futures can be implemented in python.

Futures are derived from *category theory*, a branch of mathematics that deals with abstract concepts. Futures are a type of ***Monads***. 

> I will be writing a post on monads and their varied  uses in programming shortly, where I will cover the theory behind futures as well. This post assumes you know what futures are and how they are related to monads.  If you are not familiar with monads, [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) can act as a good crash course for the same.

A future can be considered as a *state machine*. It has *two* states. *Unresolved* and *resolved*. Once resolved, a future is *immutable*. It can’t go back to being unresolved or change the value it was resolved with. A future can be resolved because an operation was successful or even when the operation was unsuccessful but complete. So, the future has *support for error handling built-in*.  

One can attach certain operations that have to be completed once the future is resolved (either successfully or unsuccessfully). I call it, *showing interest in the value of the future*.

Future is always resolved as a result of occurrences, *external to the future*. Like the arrival of a message or passing of a certain period of time. This is to say that a *future can’t change its state on its own*. So we have to arrange for a way to complete or resolve the future, when we create one. 

Now that we know enough concepts, let us have a look at how these ideas can be manifested in code. 
First, let’s define a class that represents a Future. 

```python
class Future():
    def __init__(self):
        self.pending_requests = []
        self.value = None
        self.completed = False

    def request(self, fn):
        if self.completed:
            fn(self.value)
        else:
            self.pending_requests.append(fn)

    def complete(self, value):
        if self.completed:
            raise FutureAlreadyResolved()
        self.completed = True
        self.value = value
        for fn in self.pending_requests:
            fn(self.value)
        self.pending_requests = None
```
The ***value*** and ***completed*** fields are used to provide information about the current state of the future. The ***request*** method is used to register an operation that has interest in the value that the future resolves to. The registered operations (mostly functions) are stored in the ***pending_requests*** field of the Future. 
The ***complete*** method is passed to some other part of the code, which completes the future on the occurrence of a certain event. When a future is resolved or completed, *all the operations that are registered with it are executed, passing the value that the future resolved to*. 
Now we can do things like attaching the print operation to the future using the ***request*** method on the future, which prints the value the future resolves to, once it’s resolved using the completed method from some other part of the code.

The Future works as expected, *but is useless on its own*. The true advantage of using Futures over simple event based execution (or callbacks ) is the **ability to compose concurrent operations**. Two futures, each representing a concurrent operation, can be composed to represent a bigger concurrent operation that is the combination of the two smaller concurrent operations. This bigger future can be passed around, composed with other futures, in a sequential or parallel fashion, in order to represent a much complex concurrent operation. 

Let’s start with performing a *trivial* (one that doesn’t take much time or doesn’t return another future) operation like converting the resolved value into string and prepending it with the phrase *Monkey D Luffy*. We need to create a new future that represents this transformation on the resolved value of the original future. This is similar to the ***map*** operation on arrays in JS or python. You create a new array (or modify the original) with values that is formed by applying the transformation on each of the entries of the original array. This is the idea of a *Functor in category theory ( and functional programming)*. 
> I will explain the uses of various functors in programming in an upcoming post 

So, we need to make our futures a *Functor*. We can do this by adding a method to our future *that takes in a function and gives out another future that is the result of applying this function on the resolved value of the original future*. Remember that the original future need not be already resolved when we make this connection between the newly formed future and the original one. But when the original future is resolved, we need to resolve the new future as well with the modified resolved value. 
This can be expressed in code as a method on the future, which takes in a function and gives out a new future that is resolved when the original future is resolved. This is achieved by specifying that the new future is interested in the value of the old one (using the ***request*** method on the original future).

```python
def map(self, fn):
    future = Future()
    self.request(lambda value: future.complete(fn(value)))
    return future
```        
But what if the function that we passed also returns a future? How do we deal with this? If we simply use the ***map*** method here, we will end up *with a future within another future condition*. How do we make sense of this?

If the passed function (***fn*** in the code) returns a future, then it means that we have to resolve the future that ***fn*** returned, before resolving the future:  ***future***. Yes, I know it’s confusing…
So, this can be done in code as follows:

```python
def and_then(self, fn):
    future = Future()
    def complete_new_future(value):
        new_future = fn(value)
        new_future.request(lambda value: future.complete(value))
    self.request(complete_new_future)
    return future
```
        
Here, when the original future (***self***) is completed, we run the ***fn*** function, which gives us the ***new_future***. We need our future to be resolved with the value that ***new_future*** gets resolved to. So we make a connection between the ***new_future*** and the ***future*** such that when ***new_future*** is completed (again, this should be done as a part of the ***fn*** function), it (***new_future***) completes the ***future*** as well. So there is no scope for a *future within a future scenario*. This behaviour of the futures make them a **monad**. (*Strictly speaking, there is more to a monad than just this feature. But this is enough for our purposes*)

> When a new future is created, a way to complete it should be arranged immediately

The ***map*** and ***and_then*** functions work when you already know if the passed function (***fn***) returns a future or not. (These functions are similar to the ***map*** and ***and_then*** functions used in Rust for ***Future***, ***Option*** etc.)

But what if we don’t know beforehand the nature of the passed function? JS’s then function handles both kind functions. Let’s define a similar then function ourselves, which depending on the type of the return value of the function, handles the situation aptly. 

```python
def then(self, fn):
    future = Future()
        def complete_new_future(value):
            new = fn(value)
            if isinstance(new, Future):
                new.request(lambda value: future.complete(value))
            else:
                future.complete(new)
    self.request(complete_new_future)
    return future
```




The ***isinstance*** call is used to choose whether to behave like ***map*** function or as ***and_then*** function.

So, yeah we have our own future implementation. Let’s try it out.We saw how to request a future to some operation when it’s completed and also how to get a new future from an old one.But how do we create a future in the first place? Let’s see some examples.

```python
def Unit(value):
    future = Future()
        future.complete(value)
    return future
```
 This is the simplest possible way to create a future. You just enclose a value in the future covering. This process is called **Lifting** (imagine the normal values are present in the bottom layer and the future enclosed values are present in the upper layer). You just complete the function immediately, right after you create it. 
 
```python
def Delay(value, ms):
    future = Future()
    TimePoller(ms / 1000, value, future)
    return future
    
class TimePoller():
    def __init__(self, time_in_seconds, resolve_value, future):
        self.time = time_in_seconds
        self.resolve_value = resolve_value
        self.future = future
        threading.Thread(target=self.poll).start()
    def poll(self):
        time.sleep(self.time)
        self.future.complete(self.resolve_value)
```

***Delay*** creates a future and arranges a way to complete it. This arrangement is in the form of the ***Timepoller*** object, which uses a *separate thread* to complete the future after the specified time period. 

Similarly the ***FileRead*** creates a future which is completed by a separate thread with the file contents read. 
```python
def FileRead(filename):
    future = Future()
    FilePoller(filename,future)
    return future
    
class FilePoller():
    def __init__(self, filename, future):
        self.filename = filename
        self.future = future
        threading.Thread(target=self.poll).start()
    def poll(self):
        with open(self.filename, 'r') as f:
            contents = f.read()
            self.future.complete(contents)
```

Now, you can do things like:

```python
f = Future.Delay("readme", 5000)\
    .then(lambda x: x.capitalize())\
    .then(Future.FileRead)\
    .then(lambda x: x.capitalize())
f.request(print)
```

This looks a lot like *JS Promises*. Now, if observed closely, there is *only the happy path*. What if something goes wrong while reading the file? What is the specified file is not there? So we need error handling. The error handling that I came up with is similar to the ***Result*** monad in Rust or ***Either*** in Haskell. 

```python
class Result():
    def __init__(self, value=None, error=None):
        self.value = value
        self.error = error
    def flat_map(self, fn):
        result = Result()
        if self.error:
            result.error = self.error
        else:
            v = fn(self.value)
            if isinstance(v, Result):
                if v.error:
                    result.error = v.error
                else:
                    result.value = v.value
            else:
                result.value = v
            return result
    def correct_error(self, fn):
        result = Result()
        if self.error:
            v = fn(self.error)
            if isinstance(v, Result):
                if v.error:
                    result.error = v.error
                else:
                    result.value = v.value
            else:
                result.value = v
        else:
            result.value = self.value
        return result
```      
If you observe closely, the ***Result*** class is also a *monad*. It has two ways to convert a ***Result*** object into another ***Result*** object: 
* ***correct_error*** 
* ***flat_map*** 

***flat_map*** is used for the happy path, when the ***self*** object contains ***value*** and not the ***error***. ***correct_error*** is for the unhappy path. Both take in a function that works on appropriate fields of the ***Result*** object and return a new ***Result*** object. This can be merged with our futures, by making sure that the ***Future*** is always completed with ***Result***, instead of plain values and changing our then function as follows,
```python
def then(self, fns):
    future = Future()
    x = lambda value: future.complete(Result(value=value))
    y = lambda error: future.complete(Result(error=error))
    def complete_new_future1(value):  # receives a normal value as this is called from the flat_map
        print("from the complete_new_future: before finding new")
        try:
            new = fns[0](value)
            if isinstance(new, Future):
                new.request((x,y))
            else:
                future.complete(Result(value=new))
        except Exception as e:
            future.complete(Result(error=e))


    def complete_new_future2(value):  # receives a normal value as this is called from the check_error
        try:
            new = fns[1](value)
            if isinstance(new, Future):
                new.request((x,y))
            else:
                future.complete(Result(value=new))
        except Exception as e:
            future.complete(Result(error=e))
    self.request((complete_new_future1, complete_new_future2))
    return future
```
Now you can do things like, 
```python
def id(x):
    return x
f = Future.Delay(“readme", 1000)\
    .then((lambda x: x.capitalize(), id))\
    .then((Future.FileRead,id))\
    .then((lambda x: x.split()[-1], id))\
    .then((lambda x: x.capitalize(), id))\
    .then((lambda x: x[0], id))\
    .then((id, lambda x: "there has been some kind of an error"))
f.request((print, print)) 
```  
This looks similar to *JS Promises* as well (the version where you provide both the success and error handlers to every then call). You can build on this code to implement parallels to *Promise.All* etc.

One thing I'd advise anyone who wants to do a similar thing is to use a statically typed language. The presence of a compiler that checks the type of function return values being passed into other functions makes it a lot easier to debug code.
In the next post we will see how to use the futures implemented here to design a simple ***event loop***.