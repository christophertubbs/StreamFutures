# StreamFutures
Python library implementing concurrent futures using redis streams

## Problem

Python `concurrent.futures` is rather good at distributing work across processes or threads in a way that is easy to run and get results. 
This does _not_ work particularly well for possibly long running tasks that may be communicated through Redis-based streams.

## Solution

`StreamFutures` will provide a new way to use the interfaces provided via `concurrent.futures` to distribute work via a Redis stream.

A `Future` for a Redis stream will need to submit to a stream, monitor the task's health and return when the task on the other end
either errors or returns a value. This might need to have a worker process on each listening node that will receive tasks, attach 
heartbeats to their running processes to mark a task as having failed if interuppted, and provide signal handlers to mark other interuptions
(such as `SIGTERM`) as failures.

Elements in Redis may be structured like:

```
<application>:<operation identifier>:status = {<task id>: <status enum value>}
<application>:<operation identifier>:heartbeat = {<task id>: <time since last heartbeat>}
<application>:<operation identifier>:results = {<task id>: ''}
```

An example may look like:

```
MyApp:3a1b2c3:status = {b'one': b'0', b'two': b'1', b'three': b'2', b'four': b'3', b'five': b'1'}
MyApp:3a1b2c3:heartbeat = {b'one': b'1722355139.617246', b'two': b'1722355140.94', b'three': b'1722355139.1772459', b'four': b'1722355138.172526', b'five': b'1722265296.617246'}
MyApp:3a1b2c3:results = {b'one': b'', b'two': b'', b'three': b'{"event": "process", "input": 3, "output": 5}', b'four': b'RuntimeError: This is an example of an error', b'five': b''}
```

This might be interpretted as:

### Status
* Task "one" for 3a1b2c3 in MyApp has not started
* Task "two" for 3a1b2c3 in MyApp is running
* Task "three" for 3a1b2c3 in MyApp is complete
* Task "four" for 3a1b2c3 in MyApp has encountered an error
* Task "five" for 3a1b2c3 in MyApp is running

### Heartbeat
* Task "one" for 3a1b2c3 in MyApp last proved it was running 2 seconds ago
* Task "two" for 3a1b2c3 in MyApp last proved it was running 0.04 seconds ago
* Task "three" for 3a1b2c3 in MyApp last proved it was running 2.44 seconds ago
* Task "four" for 3a1b2c3 in MyApp last proved it was running 3.44472 seconds ago
* Task "five" for 3a1b2c3 in MyApp last proved it was running 89845 seconds ago
  * Task "five" for 3a1b2c3 in MyApp has not updated recently, indicating an error has probably occurred

### Output
* Task "one" for 3a1b2c3 in MyApp has not generated output
* Task "two" for 3a1b2c3 in MyApp has not generated output
* Task "three" for 3a1b2c3 in MyApp has generated json as output
* Task "four" for 3a1b2c3 in MyApp has encountered an error
* Task "five" for 3a1b2c3 in MyApp has not generated output

  


## Responsibilities

The `Future` will be responsible for polling these three hashes. If it sees a heartbeat that is too long for a task with a status of `2`, the 
`Future` will update the status stating it has failed. The `Future` may return when all entries in `status` are `2` or `3`.

Calling `del` on the `Future` will need to halt all processes (probably expressed via the stream and via the worker listening to it), and should remove 
its entries from the redis instance unless told otherwise.

Creating a `Future` should fail if no active listeners are found. There will need to be a way to detect if a listener is currently active, probably via a heartbeat.

Calling `submit` instead of `map` should probably just create `hash`es with a single value.

## Expected Usage

Expected use should look like:

```python
import typing
from concurrent.futures import Future

from streamfutures import StreamFuturePool
from streamfutures import StreamFuture
from whatever import function
from whatever import async_function

with StreamFuturePool(workers=9, host="localhost", db=3) as pool:
    individual_value: Future = pool.submit(function, args=(3,8))
    multiple_values: typing.List[Future] = pool.map(function, *[(index, index + 1) for index in range(0, 12, 2)])
```

Each `submit` for `StreamFuturePool` will create its own `<operation identifier>` and `<task ID>` that are identical and each `map` will create its own 
`<operation identifier>` for all tasks under the `map` with each function call underneath with its own `<task ID>`.
