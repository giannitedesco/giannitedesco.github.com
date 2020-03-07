---
title: Python socket servers can drop received packets on exit
layout: post
comments: true
mathjax: true
date: 2019-06-16 10:14:00 +0900
tags: python sockets asyncio
summary: In which we discover a problem with python asyncio
---

Let's say we have a datagram server written in python using the shiny new
`asyncio` module:

```python
import socket
import asyncio
import os

class PacketCounter(asyncio.DatagramProtocol):
    def __init__(self):
        self.counter = 0
    def datagram_received(self, data, addr):
        self.counter += 1

class Writer:
    def __init__(self, socket_path, nr_msgs):
        self.sock = socket.socket(socket.AF_UNIX, socket.SOCK_DGRAM, 0)
        self.sock.connect(socket_path)
        self.sock.setblocking(False)
        self.sent_msgs = 0
        self.nr_msgs = nr_msgs
        loop.add_writer(self.sock.fileno(), self.writable)
        self.wait = loop.create_future()

    def writable(self):
        while self.sent_msgs < self.nr_msgs:
            try:
                self.sock.send(b'Hello')
                self.sent_msgs += 1
            except BlockingIOError:
                return
        if self.sent_msgs >= self.nr_msgs:
            loop.remove_writer(self.sock.fileno())
            self.wait.set_result(None)

loop = asyncio.get_event_loop()

socket_path = 'hello'

# Create the server
try:
    os.unlink(socket_path)
except FileNotFoundError:
    pass
proto = PacketCounter()
t = loop.create_datagram_endpoint(lambda :proto,
                                  family=socket.AF_UNIX,
                                  local_addr=socket_path)
transport, _ = loop.run_until_complete(t)

# Create the client and run it
sender = Writer(socket_path, 1000)
loop.run_until_complete(sender.wait)
transport.close()
loop.close()

print('tx', sender.sent_msgs)
print('rx', proto.counter)
```

If we run it, we get:
```
tx 1000
rx 841
```

Hrm...

## What's going on?
When the kernel receives a packet it gets placed in to the relevant socket's
receive buffer, the socket becomes readable potentially causing a wakeup event
in `poll()`, `epoll_wait()`, or a synchronous `read()` or `recv()`.

In the case of a datagram socket, a `recv()` will be woken up once for each
received datagram. This is how the message boundaries are preserved.

So let's say the kernel receives 3 messages for a datagram socket. The
application will need to call `recv()` 3 times to receive all of those.  Or
equivalently, sleep in `poll()` 3 times and do a non-blocking `recv()` each
time.

In any case, when it tries to read for a fourth time, it will either sleep if
the socket is a blocking socket (the default), or it will return `EAGAIN` error
(or raise `BlockingIOError` in python). There is no more data remaining in the
kernel's buffer, so the socket is not ready.

Now, in python's selector event loop (the default for UNIX-like operating
systems). When socket becomes readable, python wakes up from `epoll_wait()`
(for example), eventually calls `_SelectorDatagramTransport._read_ready` which
performs the `sock.recv()` and then calls `protocol.datagram_received()` That
will process that one single datagram. After that we'll go through the other
selector events and then go back to sleep in `epoll_wait()`.

However, since the sender has finished it's work `loop.run_until_complete()`
exits and now we try and close everything down.

But even though I gave the socket server every possible chance to grab
everything in the kernel's socket buffer (`transport.close()`, `loop.close()`).
Those remaining packets that could have finished a perilous journey accross the
world to get in to my kernel's receive buffer just get dropped to the floor like so many crumpled up cigarette boxes.

## Why this sucks
First of all, this confused me because I'm used to event frameworks in C, using
epoll, where the mainloop doesn't finish an iteration until all event sources
have hit `EAGAIN`, thereby draining the kernel's socket buffer of received data
before exiting.

In python asyncio, however, you need to be totally explicit if you want this
behaviour. But that's okay, right? These are all valid design choices,
after all.

I'm not so sure. You see, in `BaseSelectorEventLoop._read_from_self()` it looks like the right kind of behaviour happens when the self-pipe becomes readable:

```python
def _read_from_self(self):
while True:
    try:
	data = self._ssock.recv(4096)
	if not data:
	    break
	self._process_self_data(data)
    except InterruptedError:
	continue
    except BlockingIOError:
	break
```

But if I want this sort of behaviour for my datagram server I can't easily mess
around with `_SelectorDatagramTransport._read_ready`
and such-like.

I have to use some kind of workaround like this:

```python
# ... 8< ...
# Create the client and run it
sender = Writer(socket_path, 1000)
loop.run_until_complete(sender.wait)

# Now, drain the kernel socket-buffer
while True:
    try:
        buf = sock.recv(64 * 1024)
    except BlockingIOError:
        break
    proto.datagram_received(buf, None)
transport.close()

loop.close()

print('tx', sender.sent_msgs)
print('rx', proto.counter)
```

But this is going to get seriously annoying if you have multiple listening
sockets all over your program.

On the other hand, if I am worried that my program won't exit because I'll be
spending forever servicing a never-ending stream of datagrams, I think that
that's a much easier problem to solve. You just close the socket, then there's
no way to keep receiving stuff.

##  A solution
Why not use edge-triggered epoll and implement the edge-triggered behaviour
under the hood (re-try callbacks until `BlockingIOError`) without the user
having to know about it? It's going to result in way fewer system calls, and
way fewer gotchas like this. And if you want the old behaviour it's easy to
just break out of the loop immediately by calling `loop.stop()`.

There will be concerns about "starvation" but starvation can be easily avoided
by putting ready file-descriptors in to a "ready queue" and looping over the
list calling each callback in round-robin (hell, priority-queue for all I care)
fashion. Then you can just `epoll_wait(..., 0)` after some number of iterations
has been done but the ready-queue is still non-empty. That will just get you
any new fd's which became writable in the meantime and you can add those to the
end of the "ready queue."

Seriously guys... come on :)
