# psq - Cloud Pub/Sub Task Queue for Python.

[![Build Status](https://travis-ci.org/GoogleCloudPlatform/psq.svg)](https://travis-ci.org/GoogleCloudPlatform/psq) [![Coverage Status](https://coveralls.io/repos/GoogleCloudPlatform/psq/badge.svg?branch=master&service=github)](https://coveralls.io/github/GoogleCloudPlatform/psq?branch=master)

``psq`` is an example Python implementation of a simple distributed task queue using [Google Cloud Pub/Sub](https://cloud.google.com/pubsub/).

``psq`` requires minimal configuration and relies on Cloud Pub/Sub to provide scalable and reliable messaging. 

``psq`` is intentionally similar to [rq](http://python-rq.org/) and [simpleq](https://github.com/rdegges/simpleq), and takes some inspiration from [celery](http://www.celeryproject.org/) and [this blog post](http://jeffknupp.com/blog/2014/02/11/a-celerylike-python-task-queue-in-55-lines-of-code/).


## Installation

Install via [pip](https://pypi.python.org/pypi/pip):

    pip install psq

## Prerequisites

* A project on the [Google Developers Console](https://console.developers.google.com).
* The [Google Cloud SDK](https://cloud.google.com/sdk) installed locally.
* You will need the [Cloud Pub/Sub API enabled](https://console.developers.google.com/flows/enableapi?apiid=datastore,pubsub) on your project. The link will walk you through enabling the API.
* You will need to run ``gcloud auth`` before running these examples so that authentication to Google Cloud Platform services is handled transparently.

## Usage

First, create a task:

```python
def adder(a, b):
    return a + b
```

Then, create a pubsub client and a queue:

```python
from gcloud import pubsub
import psq


PROJECT_ID = 'your-project-id'

client = pubsub.Client(project=PROJECT_ID)

q = psq.Queue(client)
```

Now you can enqueue tasks:

```python
from tasks import adder

q.enqueue(adder)
```

In order to get task results, you have to configure storage:

```python
from gcloud import pubsub
import psq


PROJECT_ID = 'your-project-id'

ps_client = pubsub.Client(project=PROJECT_ID)
ds_client = datastore.Client(dataset_id=PROJECT_ID)

q = psq.Queue(
    ps_client,
    storage=psq.DatastoreStorage(ds_client))
```

With storage configured, you can get the result of a task:

```python
r = q.enqueue(adder, 5, 6)
r.result() # -> 11
```

You can also define multiple queues:

```python
fast = psq.Queue(client, 'fast')
slow = psq.Queue(client, 'slow')
```

## Things to note

Because ``psq`` is largely similar to ``rq``, similar rules around tasks apply. You can put any Python function call on a queue, provided:

 * The function is importable by the worker. This means the ``__module__`` that the funciton lives in must be importable. Notably, you can't enqueue functions that are declared in the __main__ module - such as tasks defined in a file that is run directly with ``python`` or via the interactive interpreter.
 * The worker and the applications queueing tasks must share exactly the same source code.
 * The function can't depend on global context such as global variables, current_request, etc. Pass any needed context into the worker at queue time.

### Delivery guarantees 

Pub/sub guarantees your tasks will be delivered to the workers, but ``psq`` doesn't presently guarantee that a task completes execution or exactly-once semantics, though it does allow you to provide your own mechanisms for this. This is similar to Celery's [default](http://celery.readthedocs.org/en/latest/faq.html#faq-acks-late-vs-retry) configuration.

Task completion guarantees can be provided via late ack support. Late ack is possible with Cloud Pub/sub, but it currently not implemented in this library. See [contributing.md](CONTRIBUTING.md).

There are many approaches for exactly-once semantics, such as distributed locks. This is possible in systems such as [zookeeper](http://zookeeper.apache.org/doc/r3.1.2/recipes.html#sc_recipes_Locks) and [redis](http://redis.io/topics/distlock).

## Running a worker

Execute ``psqworker`` in the *same directory where you tasks are defined*:

    psqworker.py config.q

``psqworker`` only operates on one queue at a time. If you want a server to listen to multiple queues, use something like [supervisord](http://supervisord.org/) to run multiple ``psqworker`` processes.


## Broadcast queues

A normal queue will send a single task to a single worker, spreading your tasks over all workers listening to the same queue. There are also broadcast queues, which will deliver a copy of the task to *every* worker. This is useful in situations where you want every worker to execute the same task, such as installing or upgrading software on every server.

```python
broadcast_q = psq.BroadcastQueue(client)

def restart_apache_task():
    call(["apachectl", "restart"])

broadcast_q.enqueue(restart_apache_task)
```

Broadcast queues provide an implementation of the solution described in [Reliable Task Scheduling on Google Compute Engine](https://cloud.google.com/solutions/reliable-task-scheduling-compute-engine).

*Note*: broadcast queues do not currently support any form of storage and do not support return values.

## Retries

Raising ``psq.Retry`` in your task will cause it to be retried.

```python
from psq import Retry

def retry_if_fail(self):
    try:
        r = requests.get('http://some.flaky.service.com')
    except Exception as e:
        logging.error(e)
        raise Retry()
```

## Flask & other contexts

You can bind an extra context manager to the queue.

```python
app = Flask(__name__)

q = psq.Queue(extra_context=app.app_context)
```

This will ensure that the context is available in your tasks, which is useful for things such as database connections, etc.:


```python
from flask import current_app

def flasky_task():
    backend = current_app.config['BACKEND']
```

## Ideas for improvements

* some sort of storage solution for broadcast queues.
* Memcache/redis value store.
* @task decorator that adds a delay/defer function.
* Task chaining / groups / chords.
* Late ack.
* Gevent worker.
* batch support for queueing.

## Contributing changes

* See [CONTRIBUTING.md](CONTRIBUTING.md)

## Licensing

* See [LICENSE](LICENSE)
