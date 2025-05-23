.. _guide-workers:

===============
 Workers Guide
===============

.. contents::
    :local:
    :depth: 1

.. _worker-starting:

Starting the worker
===================

.. sidebar:: Daemonizing

    You probably want to use a daemonization tool to start
    the worker in the background. See :ref:`daemonizing` for help
    starting the worker as a daemon using popular service managers.

You can start the worker in the foreground by executing the command:

.. code-block:: console

    $ celery -A proj worker -l INFO

For a full list of available command-line options see
:mod:`~celery.bin.worker`, or simply do:

.. code-block:: console

    $ celery worker --help

You can start multiple workers on the same machine, but
be sure to name each individual worker by specifying a
node name with the :option:`--hostname <celery worker --hostname>` argument:

.. code-block:: console

    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker1@%h
    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker2@%h
    $ celery -A proj worker --loglevel=INFO --concurrency=10 -n worker3@%h

The ``hostname`` argument can expand the following variables:

    - ``%h``:  Hostname, including domain name.
    - ``%n``:  Hostname only.
    - ``%d``:  Domain name only.

If the current hostname is *george.example.com*, these will expand to:

+----------+----------------+------------------------------+
| Variable | Template       | Result                       |
+----------+----------------+------------------------------+
| ``%h``   | ``worker1@%h`` | *worker1@george.example.com* |
+----------+----------------+------------------------------+
| ``%n``   | ``worker1@%n`` | *worker1@george*             |
+----------+----------------+------------------------------+
| ``%d``   | ``worker1@%d`` | *worker1@example.com*        |
+----------+----------------+------------------------------+

.. admonition:: Note for :pypi:`supervisor` users

   The ``%`` sign must be escaped by adding a second one: `%%h`.

.. _worker-stopping:

Stopping the worker
===================

Shutdown should be accomplished using the :sig:`TERM` signal.

When shutdown is initiated the worker will finish all currently executing
tasks before it actually terminates. If these tasks are important, you should
wait for it to finish before doing anything drastic, like sending the :sig:`KILL`
signal.

If the worker won't shutdown after considerate time, for being
stuck in an infinite-loop or similar, you can use the :sig:`KILL` signal to
force terminate the worker: but be aware that currently executing tasks will
be lost (i.e., unless the tasks have the :attr:`~@Task.acks_late`
option set).

Also as processes can't override the :sig:`KILL` signal, the worker will
not be able to reap its children; make sure to do so manually. This
command usually does the trick:

.. code-block:: console

    $ pkill -9 -f 'celery worker'

If you don't have the :command:`pkill` command on your system, you can use the slightly
longer version:

.. code-block:: console

    $ ps auxww | awk '/celery worker/ {print $2}' | xargs kill -9

.. versionchanged:: 5.2
    On Linux systems, Celery now supports sending :sig:`KILL` signal to all child processes
    after worker termination. This is done via `PR_SET_PDEATHSIG` option of ``prctl(2)``.

.. _worker_shutdown:

Worker Shutdown
---------------

We will use the terms *Warm, Soft, Cold, Hard* to describe the different stages of worker shutdown.
The worker will initiate the shutdown process when it receives the :sig:`TERM` or :sig:`QUIT` signal.
The :sig:`INT` (Ctrl-C) signal is also handled during the shutdown process and always triggers the 
next stage of the shutdown process.

.. _worker-warm-shutdown:

Warm Shutdown
~~~~~~~~~~~~~

When the worker receives the :sig:`TERM` signal, it will initiate a warm shutdown. The worker will
finish all currently executing tasks before it actually terminates. The first time the worker receives
the :sig:`INT` (Ctrl-C) signal, it will initiate a warm shutdown as well.

The warm shutdown will stop the call to :func:`WorkController.start() <celery.worker.worker.WorkController.start>`
and will call :func:`WorkController.stop() <celery.worker.worker.WorkController.stop>`.

- Additional :sig:`TERM` signals will be ignored during the warm shutdown process.
- The next :sig:`INT` signal will trigger the next stage of the shutdown process.

.. _worker-cold-shutdown:

Cold Shutdown
~~~~~~~~~~~~~

Cold shutdown is initiated when the worker receives the :sig:`QUIT` signal. The worker will stop
all currently executing tasks and terminate immediately.

.. _worker-REMAP_SIGTERM:

.. note::

    If the environment variable ``REMAP_SIGTERM`` is set to ``SIGQUIT``, the worker will also initiate
    a cold shutdown when it receives the :sig:`TERM` signal instead of a warm shutdown.

The cold shutdown will stop the call to :func:`WorkController.start() <celery.worker.worker.WorkController.start>`
and will call :func:`WorkController.terminate() <celery.worker.worker.WorkController.terminate>`.

If the warm shutdown already started, the transition to cold shutdown will run a signal handler ``on_cold_shutdown``
to cancel all currently executing tasks from the MainProcess and potentially trigger the :ref:`worker-soft-shutdown`.

.. _worker-soft-shutdown:

Soft Shutdown
~~~~~~~~~~~~~

.. versionadded:: 5.5

Soft shutdown is a time limited warm shutdown, initiated just before the cold shutdown. The worker will
allow :setting:`worker_soft_shutdown_timeout` seconds for all currently executing tasks to finish before
it terminates. If the time limit is reached, the worker will initiate a cold shutdown and cancel all currently
executing tasks. If the :sig:`QUIT` signal is received during the soft shutdown, the worker will cancel all
currently executing tasks but still wait for the time limit to finish before terminating, giving a chance for
the worker to perform the cold shutdown a little more gracefully.

The soft shutdown is disabled by default to maintain backward compatibility with the :ref:`worker-cold-shutdown`
behavior. To enable the soft shutdown, set :setting:`worker_soft_shutdown_timeout` to a positive float value.
The soft shutdown will be skipped if there are no tasks running. To force the soft shutdown, *also* enable the
:setting:`worker_enable_soft_shutdown_on_idle` setting.

.. warning::

    If the worker is not running any task but has ETA tasks reserved, the soft shutdown will not be initiated
    unless the :setting:`worker_enable_soft_shutdown_on_idle` setting is enabled, which may lead to task loss
    during the cold shutdown. When using ETA tasks, it is recommended to enable the soft shutdown on idle.
    Experiment which :setting:`worker_soft_shutdown_timeout` value works best for your setup to reduce the risk
    of task loss to a minimum.

For example, when setting ``worker_soft_shutdown_timeout=3``, the worker will allow 3 seconds for all currently
executing tasks to finish before it terminates. If the time limit is reached, the worker will initiate a cold shutdown
and cancel all currently executing tasks.

.. code-block:: console

    [INFO/MainProcess] Task myapp.long_running_task[6f748357-b2c7-456a-95de-f05c00504042] received
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 1/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 2/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 3/2000s
    ^C
    worker: Hitting Ctrl+C again will initiate cold shutdown, terminating all running tasks!

    worker: Warm shutdown (MainProcess)
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 4/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 5/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 6/2000s
    ^C
    worker: Hitting Ctrl+C again will terminate all running tasks!
    [WARNING/MainProcess] Initiating Soft Shutdown, terminating in 3 seconds
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 7/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 8/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 9/2000s
    [WARNING/MainProcess] Restoring 1 unacknowledged message(s)

- The next :sig:`QUIT` signal will cancel the tasks that are still running in the soft shutdown, but the worker
  will still wait for the time limit to finish before terminating.
- The next (2nd) :sig:`QUIT` or :sig:`INT` signal will trigger the next stage of the shutdown process.

.. _worker-hard-shutdown:

Hard Shutdown
~~~~~~~~~~~~~

.. versionadded:: 5.5

Hard shutdown is mostly for local or debug purposes, allowing to spam the :sig:`INT` (Ctrl-C) signal
to force the worker to terminate immediately. The worker will stop all currently executing tasks and
terminate immediately by raising a :exc:`@WorkerTerminate` exception in the MainProcess.

For example, notice the ``^C`` in the logs below (using the :sig:`INT` signal to move from stage to stage):

.. code-block:: console

    [INFO/MainProcess] Task myapp.long_running_task[7235ac16-543d-4fd5-a9e1-2d2bb8ab630a] received
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 1/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 2/2000s
    ^C
    worker: Hitting Ctrl+C again will initiate cold shutdown, terminating all running tasks!

    worker: Warm shutdown (MainProcess)
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 3/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 4/2000s
    ^C
    worker: Hitting Ctrl+C again will terminate all running tasks!
    [WARNING/MainProcess] Initiating Soft Shutdown, terminating in 10 seconds
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 5/2000s
    [WARNING/ForkPoolWorker-8] long_running_task is running, sleeping 6/2000s
    ^C
    Waiting gracefully for cold shutdown to complete...

    worker: Cold shutdown (MainProcess)
    ^C[WARNING/MainProcess] Restoring 1 unacknowledged message(s)

.. warning::

    The log ``Restoring 1 unacknowledged message(s)`` is misleading as it is not guaranteed that the message
    will be restored after a hard shutdown. The :ref:`worker-soft-shutdown` allows adding a time window just between
    the warm and the cold shutdown that improves the gracefulness of the shutdown process.

.. _worker-restarting:

Restarting the worker
=====================

To restart the worker you should send the `TERM` signal and start a new
instance. The easiest way to manage workers for development
is by using `celery multi`:

.. code-block:: console

    $ celery multi start 1 -A proj -l INFO -c4 --pidfile=/var/run/celery/%n.pid
    $ celery multi restart 1 --pidfile=/var/run/celery/%n.pid

For production deployments you should be using init-scripts or a process
supervision system (see :ref:`daemonizing`).

Other than stopping, then starting the worker to restart, you can also
restart the worker using the :sig:`HUP` signal. Note that the worker
will be responsible for restarting itself so this is prone to problems and
isn't recommended in production:

.. code-block:: console

    $ kill -HUP $pid

.. note::

    Restarting by :sig:`HUP` only works if the worker is running
    in the background as a daemon (it doesn't have a controlling
    terminal).

    :sig:`HUP` is disabled on macOS because of a limitation on
    that platform.

Automatic re-connection on connection loss to broker
====================================================

.. versionadded:: 5.3

Unless :setting:`broker_connection_retry_on_startup` is set to False,
Celery will automatically retry reconnecting to the broker after the first
connection loss. :setting:`broker_connection_retry` controls whether to automatically
retry reconnecting to the broker for subsequent reconnects.

.. versionadded:: 5.1

If :setting:`worker_cancel_long_running_tasks_on_connection_loss` is set to True,
Celery will also cancel any long running task that is currently running.

.. versionadded:: 5.3

Since the message broker does not track how many tasks were already fetched before
the connection was lost, Celery will reduce the prefetch count by the number of
tasks that are currently running multiplied by :setting:`worker_prefetch_multiplier`.
The prefetch count will be gradually restored to the maximum allowed after
each time a task that was running before the connection was lost is complete.

This feature is enabled by default, but can be disabled by setting False
to :setting:`worker_enable_prefetch_count_reduction`.

.. _worker-process-signals:

Process Signals
===============

The worker's main process overrides the following signals:

+--------------+-------------------------------------------------+
| :sig:`TERM`  | Warm shutdown, wait for tasks to complete.      |
+--------------+-------------------------------------------------+
| :sig:`QUIT`  | Cold shutdown, terminate ASAP                   |
+--------------+-------------------------------------------------+
| :sig:`USR1`  | Dump traceback for all active threads.          |
+--------------+-------------------------------------------------+
| :sig:`USR2`  | Remote debug, see :mod:`celery.contrib.rdb`.    |
+--------------+-------------------------------------------------+

.. _worker-files:

Variables in file paths
=======================

The file path arguments for :option:`--logfile <celery worker --logfile>`,
:option:`--pidfile <celery worker --pidfile>`, and
:option:`--statedb <celery worker --statedb>` can contain variables that the
worker will expand:

Node name replacements
----------------------

- ``%p``:  Full node name.
- ``%h``:  Hostname, including domain name.
- ``%n``:  Hostname only.
- ``%d``:  Domain name only.
- ``%i``:  Prefork pool process index or 0 if MainProcess.
- ``%I``:  Prefork pool process index with separator.

For example, if the current hostname is ``george@foo.example.com`` then
these will expand to:

- ``--logfile=%p.log`` -> :file:`george@foo.example.com.log`
- ``--logfile=%h.log`` -> :file:`foo.example.com.log`
- ``--logfile=%n.log`` -> :file:`george.log`
- ``--logfile=%d.log`` -> :file:`example.com.log`

.. _worker-files-process-index:

Prefork pool process index
--------------------------

The prefork pool process index specifiers will expand into a different
filename depending on the process that'll eventually need to open the file.

This can be used to specify one log file per child process.

Note that the numbers will stay within the process limit even if processes
exit or if autoscale/``maxtasksperchild``/time limits are used.  That is, the number
is the *process index* not the process count or pid.

* ``%i`` - Pool process index or 0 if MainProcess.

    Where ``-n worker1@example.com -c2 -f %n-%i.log`` will result in
    three log files:

        - :file:`worker1-0.log` (main process)
        - :file:`worker1-1.log` (pool process 1)
        - :file:`worker1-2.log` (pool process 2)

* ``%I`` - Pool process index with separator.

    Where ``-n worker1@example.com -c2 -f %n%I.log`` will result in
    three log files:

        - :file:`worker1.log` (main process)
        - :file:`worker1-1.log` (pool process 1)
        - :file:`worker1-2.log` (pool process 2)

.. _worker-concurrency:

Concurrency
===========

By default multiprocessing is used to perform concurrent execution of tasks,
but you can also use :ref:`Eventlet <concurrency-eventlet>`. The number
of worker processes/threads can be changed using the
:option:`--concurrency <celery worker --concurrency>` argument and defaults
to the number of CPUs available on the machine.

.. admonition:: Number of processes (multiprocessing/prefork pool)

    More pool processes are usually better, but there's a cut-off point where
    adding more pool processes affects performance in negative ways.
    There's even some evidence to support that having multiple worker
    instances running, may perform better than having a single worker.
    For example 3 workers with 10 pool processes each. You need to experiment
    to find the numbers that works best for you, as this varies based on
    application, work load, task run times and other factors.

.. _worker-remote-control:

Remote control
==============

.. versionadded:: 2.0

.. sidebar:: The ``celery`` command

    The :program:`celery` program is used to execute remote control
    commands from the command-line. It supports all of the commands
    listed below. See :ref:`monitoring-control` for more information.

:pool support: *prefork, eventlet, gevent, thread*, blocking:*solo* (see note)
:broker support: *amqp, redis*

Workers have the ability to be remote controlled using a high-priority
broadcast message queue. The commands can be directed to all, or a specific
list of workers.

Commands can also have replies. The client can then wait for and collect
those replies. Since there's no central authority to know how many
workers are available in the cluster, there's also no way to estimate
how many workers may send a reply, so the client has a configurable
timeout — the deadline in seconds for replies to arrive in. This timeout
defaults to one second. If the worker doesn't reply within the deadline
it doesn't necessarily mean the worker didn't reply, or worse is dead, but
may simply be caused by network latency or the worker being slow at processing
commands, so adjust the timeout accordingly.

In addition to timeouts, the client can specify the maximum number
of replies to wait for. If a destination is specified, this limit is set
to the number of destination hosts.

.. note::

    The ``solo`` pool supports remote control commands,
    but any task executing will block any waiting control command,
    so it is of limited use if the worker is very busy. In that
    case you must increase the timeout waiting for replies in the client.

.. _worker-broadcast-fun:

The :meth:`~@control.broadcast` function
----------------------------------------------------

This is the client function used to send commands to the workers.
Some remote control commands also have higher-level interfaces using
:meth:`~@control.broadcast` in the background, like
:meth:`~@control.rate_limit`, and :meth:`~@control.ping`.

Sending the :control:`rate_limit` command and keyword arguments:

.. code-block:: pycon

    >>> app.control.broadcast('rate_limit',
    ...                          arguments={'task_name': 'myapp.mytask',
    ...                                     'rate_limit': '200/m'})

This will send the command asynchronously, without waiting for a reply.
To request a reply you have to use the `reply` argument:

.. code-block:: pycon

    >>> app.control.broadcast('rate_limit', {
    ...     'task_name': 'myapp.mytask', 'rate_limit': '200/m'}, reply=True)
    [{'worker1.example.com': 'New rate limit set successfully'},
     {'worker2.example.com': 'New rate limit set successfully'},
     {'worker3.example.com': 'New rate limit set successfully'}]

Using the `destination` argument you can specify a list of workers
to receive the command:

.. code-block:: pycon

    >>> app.control.broadcast('rate_limit', {
    ...     'task_name': 'myapp.mytask',
    ...     'rate_limit': '200/m'}, reply=True,
    ...                             destination=['worker1@example.com'])
    [{'worker1.example.com': 'New rate limit set successfully'}]


Of course, using the higher-level interface to set rate limits is much
more convenient, but there are commands that can only be requested
using :meth:`~@control.broadcast`.

Commands
========

.. control:: revoke

``revoke``: Revoking tasks
--------------------------
:pool support: all, terminate only supported by prefork, eventlet and gevent
:broker support: *amqp, redis*
:command: :program:`celery -A proj control revoke <task_id>`

All worker nodes keeps a memory of revoked task ids, either in-memory or
persistent on disk (see :ref:`worker-persistent-revokes`).

.. note::

    The maximum number of revoked tasks to keep in memory can be
    specified using the ``CELERY_WORKER_REVOKES_MAX`` environment
    variable, which defaults to 50000. When the limit has been exceeded,
    the revokes will be active for 10800 seconds (3 hours) before being
    expired. This value can be changed using the
    ``CELERY_WORKER_REVOKE_EXPIRES`` environment variable.

    Memory limits can also be set for successful tasks through the
    ``CELERY_WORKER_SUCCESSFUL_MAX`` and
    ``CELERY_WORKER_SUCCESSFUL_EXPIRES`` environment variables, and
    default to 1000 and 10800 respectively.

When a worker receives a revoke request it will skip executing
the task, but it won't terminate an already executing task unless
the `terminate` option is set.

.. note::

    The terminate option is a last resort for administrators when
    a task is stuck. It's not for terminating the task,
    it's for terminating the process that's executing the task, and that
    process may have already started processing another task at the point
    when the signal is sent, so for this reason you must never call this
    programmatically.

If `terminate` is set the worker child process processing the task
will be terminated. The default signal sent is `TERM`, but you can
specify this using the `signal` argument. Signal can be the uppercase name
of any signal defined in the :mod:`signal` module in the Python Standard
Library.

Terminating a task also revokes it.

**Example**

.. code-block:: pycon

    >>> result.revoke()

    >>> AsyncResult(id).revoke()

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed')

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
    ...                    terminate=True)

    >>> app.control.revoke('d9078da5-9915-40a0-bfa1-392c7bde42ed',
    ...                    terminate=True, signal='SIGKILL')




Revoking multiple tasks
-----------------------

.. versionadded:: 3.1


The revoke method also accepts a list argument, where it will revoke
several tasks at once.

**Example**

.. code-block:: pycon

    >>> app.control.revoke([
    ...    '7993b0aa-1f0b-4780-9af0-c47c0858b3f2',
    ...    'f565793e-b041-4b2b-9ca4-dca22762a55d',
    ...    'd9d35e03-2997-42d0-a13e-64a66b88a618',
    ])


The ``GroupResult.revoke`` method takes advantage of this since
version 3.1.

.. _worker-persistent-revokes:

Persistent revokes
------------------

Revoking tasks works by sending a broadcast message to all the workers,
the workers then keep a list of revoked tasks in memory. When a worker starts
up it will synchronize revoked tasks with other workers in the cluster.

The list of revoked tasks is in-memory so if all workers restart the list
of revoked ids will also vanish. If you want to preserve this list between
restarts you need to specify a file for these to be stored in by using the `--statedb`
argument to :program:`celery worker`:

.. code-block:: console

    $ celery -A proj worker -l INFO --statedb=/var/run/celery/worker.state

or if you use :program:`celery multi` you want to create one file per
worker instance so use the `%n` format to expand the current node
name:

.. code-block:: console

    celery multi start 2 -l INFO --statedb=/var/run/celery/%n.state


See also :ref:`worker-files`

Note that remote control commands must be working for revokes to work.
Remote control commands are only supported by the RabbitMQ (amqp) and Redis
at this point.

.. control:: revoke_by_stamped_headers

``revoke_by_stamped_headers``: Revoking tasks by their stamped headers
----------------------------------------------------------------------
:pool support: all, terminate only supported by prefork and eventlet
:broker support: *amqp, redis*
:command: :program:`celery -A proj control revoke_by_stamped_headers <header=value>`

This command is similar to :meth:`~@control.revoke`, but instead of
specifying the task id(s), you specify the stamped header(s) as key-value pair(s),
and each task that has a stamped header matching the key-value pair(s) will be revoked.

.. warning::

    The revoked headers mapping is not persistent across restarts, so if you
    restart the workers, the revoked headers will be lost and need to be
    mapped again.

.. warning::

    This command may perform poorly if your worker pool concurrency is high
    and terminate is enabled, since it will have to iterate over all the running
    tasks to find the ones with the specified stamped header.

**Example**

.. code-block:: pycon

    >>> app.control.revoke_by_stamped_headers({'header': 'value'})

    >>> app.control.revoke_by_stamped_headers({'header': 'value'}, terminate=True)

    >>> app.control.revoke_by_stamped_headers({'header': 'value'}, terminate=True, signal='SIGKILL')


Revoking multiple tasks by stamped headers
------------------------------------------

.. versionadded:: 5.3

The ``revoke_by_stamped_headers`` method also accepts a list argument, where it will revoke
by several headers or several values.

**Example**

.. code-block:: pycon

    >> app.control.revoke_by_stamped_headers({
    ...    'header_A': 'value_1',
    ...    'header_B': ['value_2', 'value_3'],
    })

This will revoke all of the tasks that have a stamped header ``header_A`` with value ``value_1``,
and all of the tasks that have a stamped header ``header_B`` with values ``value_2`` or ``value_3``.

**CLI Example**

.. code-block:: console

    $ celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2

    $ celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2 --terminate

    $ celery -A proj control revoke_by_stamped_headers stamped_header_key_A=stamped_header_value_1 stamped_header_key_B=stamped_header_value_2 --terminate --signal=SIGKILL

.. _worker-time-limits:

Time Limits
===========

.. versionadded:: 2.0

:pool support: *prefork/gevent (see note below)*

.. sidebar:: Soft, or hard?

    The time limit is set in two values, `soft` and `hard`.
    The soft time limit allows the task to catch an exception
    to clean up before it is killed: the hard timeout isn't catch-able
    and force terminates the task.

A single task can potentially run forever, if you have lots of tasks
waiting for some event that'll never happen you'll block the worker
from processing new tasks indefinitely. The best way to defend against
this scenario happening is enabling time limits.

The time limit (`--time-limit`) is the maximum number of seconds a task
may run before the process executing it is terminated and replaced by a
new process. You can also enable a soft time limit (`--soft-time-limit`),
this raises an exception the task can catch to clean up before the hard
time limit kills it:

.. code-block:: python

    from myapp import app
    from celery.exceptions import SoftTimeLimitExceeded

    @app.task
    def mytask():
        try:
            do_work()
        except SoftTimeLimitExceeded:
            clean_up_in_a_hurry()

Time limits can also be set using the :setting:`task_time_limit` /
:setting:`task_soft_time_limit` settings. You can also specify time
limits for client side operation using ``timeout`` argument of
``AsyncResult.get()`` function.

.. note::

    Time limits don't currently work on platforms that don't support
    the :sig:`SIGUSR1` signal.

.. note::

    The gevent pool does not implement soft time limits. Additionally,
    it will not enforce the hard time limit if the task is blocking.


Changing time limits at run-time
--------------------------------
.. versionadded:: 2.3

:broker support: *amqp, redis*

There's a remote control command that enables you to change both soft
and hard time limits for a task — named ``time_limit``.

Example changing the time limit for the ``tasks.crawl_the_web`` task
to have a soft time limit of one minute, and a hard time limit of
two minutes:

.. code-block:: pycon

    >>> app.control.time_limit('tasks.crawl_the_web',
                               soft=60, hard=120, reply=True)
    [{'worker1.example.com': {'ok': 'time limits set successfully'}}]

Only tasks that starts executing after the time limit change will be affected.

.. _worker-rate-limits:

Rate Limits
===========

.. control:: rate_limit

Changing rate-limits at run-time
--------------------------------

Example changing the rate limit for the `myapp.mytask` task to execute
at most 200 tasks of that type every minute:

.. code-block:: pycon

    >>> app.control.rate_limit('myapp.mytask', '200/m')

The above doesn't specify a destination, so the change request will affect
all worker instances in the cluster. If you only want to affect a specific
list of workers you can include the ``destination`` argument:

.. code-block:: pycon

    >>> app.control.rate_limit('myapp.mytask', '200/m',
    ...            destination=['celery@worker1.example.com'])

.. warning::

    This won't affect workers with the
    :setting:`worker_disable_rate_limits` setting enabled.

.. _worker-max-tasks-per-child:

Max tasks per child setting
===========================

.. versionadded:: 2.0

:pool support: *prefork*

With this option you can configure the maximum number of tasks
a worker can execute before it's replaced by a new process.

This is useful if you have memory leaks you have no control over
for example from closed source C extensions.

The option can be set using the workers
:option:`--max-tasks-per-child <celery worker --max-tasks-per-child>` argument
or using the :setting:`worker_max_tasks_per_child` setting.

.. _worker-max-memory-per-child:

Max memory per child setting
============================

.. versionadded:: 4.0

:pool support: *prefork*

With this option you can configure the maximum amount of resident
memory a worker can execute before it's replaced by a new process.

This is useful if you have memory leaks you have no control over
for example from closed source C extensions.

The option can be set using the workers
:option:`--max-memory-per-child <celery worker --max-memory-per-child>` argument
or using the :setting:`worker_max_memory_per_child` setting.

.. _worker-autoscaling:

Autoscaling
===========

.. versionadded:: 2.2

:pool support: *prefork*, *gevent*

The *autoscaler* component is used to dynamically resize the pool
based on load:

- The autoscaler adds more pool processes when there is work to do,
    - and starts removing processes when the workload is low.

It's enabled by the :option:`--autoscale <celery worker --autoscale>` option,
which needs two numbers: the maximum and minimum number of pool processes:

.. code-block:: text

        --autoscale=AUTOSCALE
             Enable autoscaling by providing
             max_concurrency,min_concurrency.  Example:
               --autoscale=10,3 (always keep 3 processes, but grow to
              10 if necessary).

You can also define your own rules for the autoscaler by subclassing
:class:`~celery.worker.autoscale.Autoscaler`.
Some ideas for metrics include load average or the amount of memory available.
You can specify a custom autoscaler with the :setting:`worker_autoscaler` setting.

.. _worker-queues:

Queues
======

A worker instance can consume from any number of queues.
By default it will consume from all queues defined in the
:setting:`task_queues` setting (that if not specified falls back to the
default queue named ``celery``).

You can specify what queues to consume from at start-up, by giving a comma
separated list of queues to the :option:`-Q <celery worker -Q>` option:

.. code-block:: console

    $ celery -A proj worker -l INFO -Q foo,bar,baz

If the queue name is defined in :setting:`task_queues` it will use that
configuration, but if it's not defined in the list of queues Celery will
automatically generate a new queue for you (depending on the
:setting:`task_create_missing_queues` option).

You can also tell the worker to start and stop consuming from a queue at
run-time using the remote control commands :control:`add_consumer` and
:control:`cancel_consumer`.

.. control:: add_consumer

Queues: Adding consumers
------------------------

The :control:`add_consumer` control command will tell one or more workers
to start consuming from a queue. This operation is idempotent.

To tell all workers in the cluster to start consuming from a queue
named "``foo``" you can use the :program:`celery control` program:

.. code-block:: console

    $ celery -A proj control add_consumer foo
    -> worker1.local: OK
        started consuming from u'foo'

If you want to specify a specific worker you can use the
:option:`--destination <celery control --destination>` argument:

.. code-block:: console

    $ celery -A proj control add_consumer foo -d celery@worker1.local

The same can be accomplished dynamically using the :meth:`@control.add_consumer` method:

.. code-block:: pycon

    >>> app.control.add_consumer('foo', reply=True)
    [{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]

    >>> app.control.add_consumer('foo', reply=True,
    ...                          destination=['worker1@example.com'])
    [{u'worker1.local': {u'ok': u"already consuming from u'foo'"}}]


By now we've only shown examples using automatic queues,
If you need more control you can also specify the exchange, routing_key and
even other options:

.. code-block:: pycon

    >>> app.control.add_consumer(
    ...     queue='baz',
    ...     exchange='ex',
    ...     exchange_type='topic',
    ...     routing_key='media.*',
    ...     options={
    ...         'queue_durable': False,
    ...         'exchange_durable': False,
    ...     },
    ...     reply=True,
    ...     destination=['w1@example.com', 'w2@example.com'])


.. control:: cancel_consumer

Queues: Canceling consumers
---------------------------

You can cancel a consumer by queue name using the :control:`cancel_consumer`
control command.

To force all workers in the cluster to cancel consuming from a queue
you can use the :program:`celery control` program:

.. code-block:: console

    $ celery -A proj control cancel_consumer foo

The :option:`--destination <celery control --destination>` argument can be
used to specify a worker, or a list of workers, to act on the command:

.. code-block:: console

    $ celery -A proj control cancel_consumer foo -d celery@worker1.local


You can also cancel consumers programmatically using the
:meth:`@control.cancel_consumer` method:

.. code-block:: console

    >>> app.control.cancel_consumer('foo', reply=True)
    [{u'worker1.local': {u'ok': u"no longer consuming from u'foo'"}}]

.. control:: active_queues

Queues: List of active queues
-----------------------------

You can get a list of queues that a worker consumes from by using
the :control:`active_queues` control command:

.. code-block:: console

    $ celery -A proj inspect active_queues
    [...]

Like all other remote control commands this also supports the
:option:`--destination <celery inspect --destination>` argument used
to specify the workers that should reply to the request:

.. code-block:: console

    $ celery -A proj inspect active_queues -d celery@worker1.local
    [...]


This can also be done programmatically by using the
:meth:`~celery.app.control.Inspect.active_queues` method:

.. code-block:: pycon

    >>> app.control.inspect().active_queues()
    [...]

    >>> app.control.inspect(['worker1.local']).active_queues()
    [...]

.. _worker-inspect:

Inspecting workers
==================

:class:`@control.inspect` lets you inspect running workers. It
uses remote control commands under the hood.

You can also use the ``celery`` command to inspect workers,
and it supports the same commands as the :class:`@control` interface.

.. code-block:: pycon

    >>> # Inspect all nodes.
    >>> i = app.control.inspect()

    >>> # Specify multiple nodes to inspect.
    >>> i = app.control.inspect(['worker1.example.com',
                                'worker2.example.com'])

    >>> # Specify a single node to inspect.
    >>> i = app.control.inspect('worker1.example.com')

.. _worker-inspect-registered-tasks:

Dump of registered tasks
------------------------

You can get a list of tasks registered in the worker using the
:meth:`~celery.app.control.Inspect.registered`:

.. code-block:: pycon

    >>> i.registered()
    [{'worker1.example.com': ['tasks.add',
                              'tasks.sleeptask']}]

.. _worker-inspect-active-tasks:

Dump of currently executing tasks
---------------------------------

You can get a list of active tasks using
:meth:`~celery.app.control.Inspect.active`:

.. code-block:: pycon

    >>> i.active()
    [{'worker1.example.com':
        [{'name': 'tasks.sleeptask',
          'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
          'args': '(8,)',
          'kwargs': '{}'}]}]

.. _worker-inspect-eta-schedule:

Dump of scheduled (ETA) tasks
-----------------------------

You can get a list of tasks waiting to be scheduled by using
:meth:`~celery.app.control.Inspect.scheduled`:

.. code-block:: pycon

    >>> i.scheduled()
    [{'worker1.example.com':
        [{'eta': '2010-06-07 09:07:52', 'priority': 0,
          'request': {
            'name': 'tasks.sleeptask',
            'id': '1a7980ea-8b19-413e-91d2-0b74f3844c4d',
            'args': '[1]',
            'kwargs': '{}'}},
         {'eta': '2010-06-07 09:07:53', 'priority': 0,
          'request': {
            'name': 'tasks.sleeptask',
            'id': '49661b9a-aa22-4120-94b7-9ee8031d219d',
            'args': '[2]',
            'kwargs': '{}'}}]}]

.. note::

    These are tasks with an ETA/countdown argument, not periodic tasks.

.. _worker-inspect-reserved:

Dump of reserved tasks
----------------------

Reserved tasks are tasks that have been received, but are still waiting to be
executed.

You can get a list of these using
:meth:`~celery.app.control.Inspect.reserved`:

.. code-block:: pycon

    >>> i.reserved()
    [{'worker1.example.com':
        [{'name': 'tasks.sleeptask',
          'id': '32666e9b-809c-41fa-8e93-5ae0c80afbbf',
          'args': '(8,)',
          'kwargs': '{}'}]}]


.. _worker-statistics:

Statistics
----------

The remote control command ``inspect stats`` (or
:meth:`~celery.app.control.Inspect.stats`) will give you a long list of useful (or not
so useful) statistics about the worker:

.. code-block:: console

    $ celery -A proj inspect stats

For the output details, consult the reference documentation of :meth:`~celery.app.control.Inspect.stats`.

Additional Commands
===================

.. control:: shutdown

Remote shutdown
---------------

This command will gracefully shut down the worker remotely:

.. code-block:: pycon

    >>> app.control.broadcast('shutdown') # shutdown all workers
    >>> app.control.broadcast('shutdown', destination='worker1@example.com')

.. control:: ping

Ping
----

This command requests a ping from alive workers.
The workers reply with the string 'pong', and that's just about it.
It will use the default one second timeout for replies unless you specify
a custom timeout:

.. code-block:: pycon

    >>> app.control.ping(timeout=0.5)
    [{'worker1.example.com': 'pong'},
     {'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

:meth:`~@control.ping` also supports the `destination` argument,
so you can specify the workers to ping:

.. code-block:: pycon

    >>> ping(['worker2.example.com', 'worker3.example.com'])
    [{'worker2.example.com': 'pong'},
     {'worker3.example.com': 'pong'}]

.. _worker-enable-events:

.. control:: enable_events
.. control:: disable_events

Enable/disable events
---------------------

You can enable/disable events by using the `enable_events`,
`disable_events` commands. This is useful to temporarily monitor
a worker using :program:`celery events`/:program:`celerymon`.

.. code-block:: pycon

    >>> app.control.enable_events()
    >>> app.control.disable_events()

.. _worker-custom-control-commands:

Writing your own remote control commands
========================================

There are two types of remote control commands:

- Inspect command

    Does not have side effects, will usually just return some value
    found in the worker, like the list of currently registered tasks,
    the list of active tasks, etc.

- Control command

    Performs side effects, like adding a new queue to consume from.

Remote control commands are registered in the control panel and
they take a single argument: the current
:class:`!celery.worker.control.ControlDispatch` instance.
From there you have access to the active
:class:`~celery.worker.consumer.Consumer` if needed.

Here's an example control command that increments the task prefetch count:

.. code-block:: python

    from celery.worker.control import control_command

    @control_command(
        args=[('n', int)],
        signature='[N=1]',  # <- used for help on the command-line.
    )
    def increase_prefetch_count(state, n=1):
        state.consumer.qos.increment_eventually(n)
        return {'ok': 'prefetch count incremented'}

Make sure you add this code to a module that is imported by the worker:
this could be the same module as where your Celery app is defined, or you
can add the module to the :setting:`imports` setting.

Restart the worker so that the control command is registered, and now you
can call your command using the :program:`celery control` utility:

.. code-block:: console

    $ celery -A proj control increase_prefetch_count 3

You can also add actions to the :program:`celery inspect` program,
for example one that reads the current prefetch count:

.. code-block:: python

    from celery.worker.control import inspect_command

    @inspect_command()
    def current_prefetch_count(state):
        return {'prefetch_count': state.consumer.qos.value}


After restarting the worker you can now query this value using the
:program:`celery inspect` program:

.. code-block:: console

    $ celery -A proj inspect current_prefetch_count
