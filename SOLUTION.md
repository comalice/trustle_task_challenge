# Trustle Scheduler Challenge

So I've had a workflow engine in my head for a few months now for other purposes. The "Scheduler" is a subset of the workflow engine's functionality. This should be fun!

## Overview

The spec given is pretty clear, so let's talk implementation details right away.

- **REST API**: The folks at Trustle like FastAPI, I like FastAPI, so we'll use FastAPI. Though, the actual API engine will be a thin reference to all our business logic, thankfully. No business logic in the API implementation.

- **Task Engine**: This is where the fun is at.

  - *Tasks:* Python already has a bunch of ways to handle async tasks. We could use `threading`, or we could use `subprocess` (with `deque`s and/or OS `Pipe`s depending on how much data we need to shove through). OR we could make everything async and let Python's built in `asyncio` handle all the brain melting goodness that is multithreading. I'm not a fan of colored functions, but we'll manage. Given this, all tasks will be functions or class methods (if we need data to persist across calls, like in the counter task).

    ```python
    # So a task looks something like this:
    async def my_excellent_task(_in: TInput, out: TOutput):
        sleep(2)
        return
    ```

    - *Task IDs*: We must differentiate between a `Task` and a "task instance"; a `Task` in code is a __definition__, a running instance of a task is... well, it's a "task instance". Each task instance will be issued a URI. We can use Python's `uuid.uuid4()` for this.

      ```pyt
      
      ```

      We'll also need to keep track of task instances, which leads to our next piece:

    - *Task In Progress Manager (TIPMan):* We'll need a directory of all task instances. We can use a database, and for simplicity we'll use Python's bundled SQLite. Every time a task starts, we'll generate a new UUID, associate it with the task, and add it to the DB. When the app gets a query for a URI, it will be handled by a simple "Running or Not" router will look first in memory, then in the DB for a given UUID. If the task is found in memory, it is running, otherwise if it is in the DB it is not running, finally, if it isn't in the DB, it never existed.

    - *Scheduler:* We'll need a way to trigger tasks on a given tempo. Fortunately for us, Python's `threading` module provides `Timer`. For timed tasks, I will assume that if a previous task hasn't finished, the next execution should be skipped. IE, a task is scheduled every five minutes, but the previous run is still going when the timer triggers, we skip the next iteration. We'll need a way to check for previously running instances of a `Task`. Ah, and we'll need a store for scheduled tasks (the "Persistent Scheduling" requirement). We'll need a transparent way to keep up with these. We'll probably put it in the DB.

  - *Events and EventBus:* I'm introducing this nomenclature from [StateCharts](https://statecharts.dev/glossary/event.html) mostly because it is useful. We'll use `Event`s to 'make stuff happen', and we'll use a super simple synchronous `EventBus` to propagate those changes into our task engine. The bus will receive events on a queue and propagate the events in order it receives them.

  ### Other Notes

I'd like to not have state stored in two or more places at once, so we're going to make use of an ORM. I'll use `Peewee` as it is what I'm familiar with. This will allow us to query our backing DB on the fly without worrying about in-memory objects vs. in-DB objects getting out of sync.

We'll also be packing this up as an executable using `pyinstaller` because reasons (you don't know Python if you don't know Python dependency hell).