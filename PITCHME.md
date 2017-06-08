## Introduction to
# Async Python

---

You are:

- interested in async python
- either unfamiliar with async, or still not fully comfortable with async python

---

I am:

- aki@mozilla.com
- aki on IRC/Slack

---

First, a story!

---

You have a home improvement project!

- remove all the old nails in the walls         |
- patch the holes in the fence                  |
- hang some art on the wall                     |

---

You have some friends willing to help!

But only one hammer.  :(

---

Esmeralda wants to fix the fence!  She will:

- walk to the pile of boards                    |
- bring a board to the next hole in the fence   |
- nail it in                                    |
- repeat                                        |

---

Hammering the nails in takes a few seconds.

Esmeralda will be spending most of her time walking to and from the pile of boards.

---

Reginald wants to remove old nails in the wall!  He will:

- find an old nail in the wall                  |
- remove it                                     |
- repeat                                        |

---

Removing the nail takes a few seconds.

Reginald will be spending most of his time finding the nails.

---

You're going to hang some art!  You will:

- find a place for the art      |
- hammer the nail in            |
- hang the art                  |
- repeat                        |

---

Hammering the nail takes a few seconds.

You will be spending most of your time finding places for the art.

---

In a synchronous model, you're going to take turns holding the hammer.

Esmeralda will fix the fence, then Reginald will remove the old nails, then you will hang the art.

Each of you will be holding the hammer while doing non-hammer-related things, even if someone else needs it.

---

In an asynchronous model, you have a magic hammer!

It flies to the next person waiting for it.

But only when the person holding it says the magic word: `await`.

---

<ul>
<li>Esmeralda gets the hammer.</li>
<li class="fragment">Esmeralda hammers the board into the fence.</li>
<li class="fragment">then she says "`await get_board()`"</li>
<li class="fragment">the hammer flies to the next person waiting.</li>
<li class="fragment">while she's getting the board, someone else can use the hammer.</li>
</ul>

---

<ul>
<li>Reginald gets the hammer!</li>
<li class="fragment">he removes the nail he found</li>
<li class="fragment">then he says "`await find_next_nail()`"</li>
<li class="fragment">the hammer flies to the next person waiting.</li>
<li class="fragment">while he finds the next nail, someone else can use the hammer.</li>
</ul>

---

<ul>
<li>You get the hammer!</li>
<li class="fragment">you hammer a nail in and hang the art.</li>
<li class="fragment">you say "`await find_next_location()`"</li>
<li class="fragment">the hammer flies to the next person waiting.</li>
<li class="fragment">while you find the location for the next piece, someone else can use the hammer.</li>
</ul>

---

Each worker hands back control of the hammer when they're not actively using it.

The entire project happens faster as a result.

---

**Hammer time.**

---

The magic hammer represents python's event loop.

The tasks which don't require the hammer represent I/O.

Only one worker can have the hammer at a time, but you can perform I/O in the background.

And each worker decides when they give up control of the hammer, using the magic word.

---

This is different from python *threads*, where the hammer changes hands automatically  and invisibly.  That means they're faster, but have the chance to interfere with each other in unexpected ways.

---

Esmeralda's work is separate, but you may be about to hang art when invisible-Reginald decides to remove that nail.  He removes it right before you hang the art, resulting in a big crash.

---

This is why safe threading is hard.

Async gets most of the benefits without as much complexity.

---

### Code Examples

Let's print out "Hello World" with some sleeps.

---

py2 and py3 synchronous

```python
from __future__ import print_function
import time

def sleep_then_print(interval, message):
    time.sleep(interval)
    print(message)

sleep_then_print(.5, "Hello")
sleep_then_print(.5, "World")
sleep_then_print(.5, "Love, Synchronous Python")
```

@[1](Let's make our code python3 compatible)
@[2](We need `time` for `time.sleep`)
@[4-6](Sleep interval, then print message)
@[8](First sleep .5 seconds, then print "Hello")
@[9](Next sleep .5 seconds, then print "World")
@[10](Finally sleep .5 seconds, then print "Love, Synchronous Python")

---

py3 asyncio, but with synchronous logic

```python
import asyncio

async def sleep_then_print(interval, message):
    await asyncio.sleep(interval)
    print(message)

loop = asyncio.get_event_loop()
loop.run_until_complete(sleep_then_print(.5, "Hello"))
loop.run_until_complete(sleep_then_print(.5, "World"))
loop.run_until_complete(sleep_then_print(.5, "Love, Asynchronous Python"))
```

@[1](We need `asyncio` for the event loop and `asyncio.sleep`)
@[3](`async def` means this is an async coroutine)
@[4](Sleep and pass the hammer to someone who can use it. We can only `await` in a coroutine)
@[5](Then print our message)
@[7](The event loop primes our magic hammer)
@[8-10](We're still using synchronous logic. We can make this concurrent)

---

py3 asyncio, concurrent logic

```python
import asyncio

async def sleep_then_print(interval, message):
    await asyncio.sleep(interval)
    print(message)

loop = asyncio.get_event_loop()
tasks = [
    asyncio.ensure_future(sleep_then_print(1, "World")),
    asyncio.ensure_future(sleep_then_print(1.5, "Love, Asynchronous Python")),
    asyncio.ensure_future(sleep_then_print(.5, "Hello")),
]
loop.run_until_complete(asyncio.wait(tasks))
```

@[8-12](Let's gather all the `sleep_then_print` calls in `tasks`, without running them yet)
@[9-11](They're intentionally out of order)
@[9](wrap the coroutine in a Future, which is an object that will have a result in the future)
@[13](`asyncio.wait` makes sure all of the `tasks` run and finish; run this in the event loop)
@[3-4](Each task enters `sleep_then_print`, and sleeps. The first ready task gets the hammer)
@[11](After .5 seconds, "Hello" is ready, and prints)
@[9](After 1 second, "World" is ready, and prints)
@[10](After 1.5 seconds, "Love, Asynchronous Python" is ready, and prints)

---

Gotchas

```python
import asyncio

async def sleep_then_print(interval, message):
    asyncio.sleep(interval)  # no
    if message:
        print(message)
    else:
        raise Exception("No message!")
    return "blah"

loop = asyncio.get_event_loop()
tasks = [
    asyncio.ensure_future(sleep_then_print(1, "World")),
    asyncio.ensure_future(sleep_then_print(1.5, "Love, Asynchronous Python")),
    asyncio.ensure_future(sleep_then_print(.5, "Hello")),
]
loop.run_until_complete(asyncio.wait(tasks))
```

@[4](If you don't `await` the `asyncio.sleep`, it doesn't do anything!)
@[8](If a future hits an exception, it's stored in the future object)
@[9](If a future gets a return value, it's stored in the future object)
@[17](The results and exceptions are now stored in `tasks`; we need to operate on them)

---

If you have return values or exceptions in your futures,

```python
results = []
# tasks are already run in this example
for task in tasks:
    exc = task.exception()
    if exc is not None:
        raise exc
    results.append(task.result())
```

@[4](retrieve their exception, if any)
@[7](retrieve their result, if any)
@[1-7](Futures also take a callback)

---

```python
def foo():
    await bar()  # no

async def bar():
    yield baz  # no
```

@[1-2](You can only await in a coroutine)
@[4-5](You can't `yield` in a py35 `async def`)

---

`loop.run_forever()` vs `loop.run_until_complete`

The former is more suited for servers that respond to events

I prefer the latter in a `while` loop for clients.

An exception in a task can leave the `run_forever` loop running forever without anything to run.

---

## Coding challenge!

---

py2 and py3 synchronous

```python
from __future__ import print_function
import requests

URLS = [...]

for url in URLS:
    req = requests.get(url, timeout=60)
    req.raise_for_status()
    print(req.json())
```

@[4](We have some urls we want to download!)
@[2](Use `requests` for synchronous downloads)
@[6-7](Synchronous: step through the urls and download one by one)
@[8](Raise an exception on errors)
@[9](Do something with the result. Just `print` for now)

---

Your turn!

Let's write an async version, using [aiohttp](http://aiohttp.readthedocs.io/en/stable/)
