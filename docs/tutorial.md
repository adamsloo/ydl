# An Introduction to YDL

## What is YDL?

YDL (pronounced "yodel") is a simple inter-process communication framework. Processes that want to communicate have _clients_ (usually one per process), and they can talk to each other through a common _server_. The server may be run on its own, or as part of one of the processes.

Within each server are several _channels_. Each channel is a unique string identifier, which determines where messages are forwarded. Any client may send a message across any channel, and it will be received by all clients listening on that channel. If there are no clients listening to a given channel, the message is simply discarded.

## A minimal example

(Prerequisites: having python3 and ydl installed)

The smallest useful example is one Python process sending a message to another Python process. For this, you will need to open three terminal windows. Execute the following command in terminal 1:
```
$ python3 -m ydl
```
This command starts the YDL server, which the two other processes will connect to. It's not important to run this command first; if you start the other two processes first, they would simply wait for the server to become availible until you ran this command.

Execute the following in terminal 2:
```
$ python3
>>> import ydl
>>> yc = ydl.YDLClient("cheese")
>>> yc.receive()
```
You'll notice that after executing the last line, the interpreter will seem to freeze. This is good! The call to `yc.receive()` will wait for the next message across any of the channels that `yc` is listening on. Currently, `yc` is listening to the channel `"cheese"`.

Execute the following in terminal 3:
```
$ python3
>>> import ydl
>>> yc = ydl.YDLClient()
>>> yc.send(("cheese", 1, 2, 3, "cool"))
```
After executing `yc.send()`, you should notice the full tuple pop up in terminal 2. Congratulations, you've sent your first message!

You should try some other `send()` calls; it can send any tuple whose elements are json serializable. The first element of the tuple must be the channel you want to send to; in this case, `"cheese"`.

After calling `send()` a bunch of times, you can call `receive()` in terminal 2 the same number of times to retrieve the messages.

## A less minimal example

The minimal example involves two processes talking to each other on the same computer; however, YDL can do much more than that. In this less minimal example, we'll demonstrate:
 - running the YDL server as a thread on one of the processes
 - communication between two processes on different computers
 - two-way communication

For this example, you will need two terminal windows on seperate computers. You may run both terminals on the same computer, but it's more interesting if you run them on two seperate computers on the same local network.

In terminal 1, execute:
```
$ python3
>>> import threading
>>> import ydl
>>> def run_server_locally():
...     ydl.run_ydl_server("0.0.0.0", 5001)
... 
>>> threading.Thread(target=run_server_locally).start()
>>> yc = ydl.YDLClient("potato", "banana")
>>> while True:
...     m = yc.receive()
...     yc.send(("cheese",) + m[1:])
```
There are a few interesting things here. First, instead of running the YDL server as its own process, it's being run as a thread on this process. Secondly, the server is listening on `0.0.0.0`, rather than the default `127.0.0.1` (this will allow it to accept connections for the local area network). Finally, the while loop means that the `yc` client will listen for any messages to `"potato"` or `"banana"` and forward them to `"cheese"`.

Now, determine the local IP address of the first computer (this can be done with `ip addr` on Linux, or by looking at the network preferences on Mac). In terminal 2, execute
```
$ python3
>>> import ydl
>>> yc = ydl.YDLClient("cheese", socket_address=("COMPUTER_1_IP_HERE", 5001))
>>> yc.send(("potato", 1234))
>>> yc.receive()
```  
(make sure to replace `"COMPUTER_1_IP_HERE"` with the IP address of your first computer, or `"127.0.0.1"` if you're running both terminal windows on the same computer)

You should see the message `('cheese', 1234)` received back. If so, congratulations! You've successfully had two processes communicate across a network. 

## Structured Communication

By default, messages are very permissive - you can pretty much send any tuple that begins with a channel name. However, such flexible comminication can become unwieldy for larger projects.

One common use case of YDL is for remote procedure calls; basically, we want to invoke some function on the receiving process. For example, we may have two processes that do something like this (make sure to run `python3 -m ydl` in a 3rd terminal if you want to run this demo):

Process 1:
```python
import ydl
yc = ydl.YDLClient("interactive")
while True:
    num = int(input("Enter an integer: "))
    op = input("Enter i to increment, or d to double: ")
    if op == "i":
        yc.send(("calculator", "i", {"num": num}))
    elif op == "d":
        yc.send(("calculator", "d", {"num": num}))
    else:
        print("unsupported operation")
        continue
    print("result: ", yc.receive()[1])
```
Process 2:
```python
import ydl

def increment(num):
    return num + 1

def double(num):
    return num * 2

yc = ydl.YDLClient("calculator")
fn_mapping = {"i": increment, "d": double}
while True:
    _, op, data = yc.receive()
    yc.send(("interactive", fn_mapping[op](**data)))
```
Here, we have process 1 doing remote procedure calls supported by process 2. However, there are some clunky bits here: 
 - function names and arguments are identified by strings, which is just inviting misspellings
 - whenever we want to send a message from process 1, we have to remember all the arguments
 - no autocompletion :(
 - no type checking

All of these problems can be solved through the use of _header functions_, which are a mechanism for creating structured messages. Let's modify the previous example:

First, a new file called `shared.py`:
```python
import ydl

calc_channel = "calculator"
int_channel = "interactive"

@ydl.header(calc_channel, "i")
def increment_message(num: int):
    pass

@ydl.header(calc_channel, "d")
def double_message(num: int):
    pass

@ydl.header(int_channel, "result")
def result_message(num: int):
    pass
```
Process 1:
```python
import ydl
from shared import *
yc = ydl.YDLClient(int_channel)
while True:
    num = int(input("Enter an integer: "))
    op = input("Enter i to increment, or d to double: ")
    if op == "i":
        yc.send(increment_message(num=num))
    elif op == "d":
        yc.send(double_message(num=num))
    else:
        print("unsupported operation")
        continue
    print("result: ", yc.receive()[2]['num'])
```
Process 2:
```python
import ydl
from shared import *

def increment(num):
    return num + 1

def double(num):
    return num * 2

yc = ydl.YDLClient(calc_channel)
fn_mapping = {
    increment_message.name: increment, 
    double_message.name: double
}
while True:
    _, op, data = yc.receive()
    yc.send(result_message(num=fn_mapping[op](**data)))
```

The annotion `@ydl.header` automatically replaces the function with one that will construct a message. The new function will do the following:
 - make sure all arguments are keyword, not positional (makes contents of messages more explicit)
 - typecheck all the arguments
 - call the original function, which has the opportunity to raise errors. For example, the original function might do bounds checks on the arguments.
 - finally, return the tuple (target_channel, header, {args}). Usually "header" corresponds to a function name.

The new function also gets new properties/variables: `fn.target` and `fn.header`, which correspond to the target channel and the header respectively. This allows the receiving end to avoid hardcoding the names.
 
## Technical Behavior

Note that YDL operates on a many-to-many messaging model, so you can have several processes listen on the same channel if needed. This might be useful for logging; you can have a logging process listen to all your channels, without changing the behavior of your application at all.

A client may listen to any number of channels, as long as they're all passed as arguments to the constructor.

Clients will connect to the server in the `YDLClient()` constructor; note that this will block if the server isn't availible. Both `send()` and `receive()` will block and try to reconnect if connection is lost.

If the server goes down, it may simply be restarted without too much chaos. Some messages may be lost if they were sent just before the server went down, or just after it comes back up. This is somewhat unavoidable, so if you need to guarentee that a message was sent, you should implement confirmation messages.



