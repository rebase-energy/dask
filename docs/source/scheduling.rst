Scheduling
==========

All of the large-scale Dask collections like
:doc:`Dask Array <array>`, :doc:`Dask DataFrame <dataframe>`, and :doc:`Dask Bag <bag>`
and the fine-grained APIs like :doc:`delayed <delayed>` and :doc:`futures <futures>`
generate task graphs where each node in the graph is a normal Python function
and edges between nodes are normal Python objects
that are created by one task as outputs and used as inputs in another task.
After Dask generates these task graphs, it needs to execute them on parallel hardware.
This is the job of a *task scheduler*.
Different task schedulers exist, and each will consume a task graph and compute the
same result, but with different performance characteristics.

Dask has two families of task schedulers:

1.  **Single-machine scheduler**: This scheduler provides basic features on a
    local process or thread pool.  This scheduler was made first and is the
    default.  It is simple and cheap to use, although it can only be used on
    a single machine and does not scale
2.  **Distributed scheduler**: This scheduler is more sophisticated, offers
    more features, but also requires a bit more effort to set up.  It can
    run locally or distributed across a cluster

|

.. image:: images/dask-overview-schedulers.svg
   :alt: Dask is composed of three parts. "Collections" create "Task Graphs" which are then sent to the "Scheduler" for execution. There are two types of schedulers that are described in more detail below.
   :align: center
   :scale: 135%

|

For different computations you may find better performance with particular scheduler settings.
This document helps you understand how to choose between and configure different schedulers,
and provides guidelines on when one might be more appropriate.

.. _threaded-scheduler:

Local Threads
-------------

.. code-block:: python

   import dask
   dask.config.set(scheduler='threads')  # overwrite default with threaded scheduler

The threaded scheduler executes computations with a local
``concurrent.futures.ThreadPoolExecutor``.
It is lightweight and requires no setup.
It introduces very little task overhead (around 50us per task)
and, because everything occurs in the same process,
it incurs no costs to transfer data between tasks.
However, due to Python's Global Interpreter Lock (GIL),
this scheduler only provides parallelism when your computation is dominated by non-Python code,
as is primarily the case when operating on numeric data in NumPy arrays, Pandas DataFrames,
or using any of the other C/C++/Cython based projects in the ecosystem.

The threaded scheduler is the default choice for
:doc:`Dask Array <array>`, :doc:`Dask DataFrame <dataframe>`, and :doc:`Dask Delayed <delayed>`.
However, if your computation is dominated by processing pure Python objects
like strings, dicts, or lists,
then you may want to try one of the process-based schedulers below
(we currently recommend the distributed scheduler on a local machine).


Local Processes
---------------

.. note::

   The :ref:`distributed scheduler <local distributed>` described below is often a better choice today.
   We encourage readers to continue reading after this section.

.. tip::

   Be sure to include an ``if __name__ == "__main__":`` block when using the multiprocessing scheduler
   in a standalone Python script. See `Standalone Python scripts`_ for more details.

.. code-block:: python

   import dask
   dask.config.set(scheduler='processes')  # overwrite default with multiprocessing scheduler


The multiprocessing scheduler executes computations with a local
``concurrent.futures.ProcessPoolExecutor``.
It is lightweight to use and requires no setup.
Every task and all of its dependencies are shipped to a local process,
executed, and then their result is shipped back to the main process.
This means that it is able to bypass issues with the GIL and provide parallelism
even on computations that are dominated by pure Python code,
such as those that process strings, dicts, and lists.

However, moving data to remote processes and back can introduce performance penalties,
particularly when the data being transferred between processes is large.
The multiprocessing scheduler is an excellent choice when workflows are relatively linear,
and so does not involve significant inter-task data transfer
as well as when inputs and outputs are both small, like filenames and counts.

This is common in basic data ingestion workloads,
such as those are common in :doc:`Dask Bag <bag>`,
where the multiprocessing scheduler is the default:

.. code-block:: python

   >>> import dask.bag as db
   >>> db.read_text('*.json').map(json.loads).pluck('name').frequencies().compute()
   {'alice': 100, 'bob': 200, 'charlie': 300}

For more complex workloads,
where large intermediate results may be depended upon by multiple downstream tasks,
we generally recommend the use of the distributed scheduler on a local machine.
The distributed scheduler is more intelligent about moving around large intermediate results.

.. _single-threaded-scheduler:

Single Thread
-------------

.. code-block:: python

   import dask
   dask.config.set(scheduler='synchronous')  # overwrite default with single-threaded scheduler

The single-threaded synchronous scheduler executes all computations in the local thread
with no parallelism at all.
This is particularly valuable for debugging and profiling,
which are more difficult when using threads or processes.

For example, when using IPython or Jupyter notebooks, the ``%debug``, ``%pdb``, or ``%prun`` magics
will not work well when using the parallel Dask schedulers
(they were not designed to be used in a parallel computing context).
However, if you run into an exception and want to step into the debugger,
you may wish to rerun your computation under the single-threaded scheduler
where these tools will function properly.


.. _local distributed:

Dask Distributed (local)
------------------------

.. tip::

   Be sure to include an ``if __name__ == "__main__":`` block when using the local distributed scheduler
   in a standalone Python script. See `Standalone Python scripts`_ for more details.

.. code-block:: python

   from dask.distributed import Client
   client = Client()
   # or
   client = Client(processes=False)

The Dask distributed scheduler can either be :doc:`setup on a cluster <deploying>`
or run locally on a personal machine.  Despite having the name "distributed",
it is often pragmatic on local machines for a few reasons:

1.  It provides access to asynchronous API, notably :doc:`Futures <futures>`
2.  It provides a diagnostic dashboard that can provide valuable insight on
    performance and progress
3.  It handles data locality with more sophistication, and so can be more
    efficient than the multiprocessing scheduler on workloads that require
    multiple processes

You can read more about using the Dask distributed scheduler on a single machine in
:doc:`these docs <deploying>`.


Dask Distributed (Cluster)
--------------------------

You can also run Dask on a distributed cluster.
There are a variety of ways to set this up depending on your cluster.
We recommend referring to :doc:`how to deploy Dask clusters <deploying>` for more information.

.. _scheduling-configuration:

Configuration
-------------

You can configure the global default scheduler by using the ``dask.config.set(scheduler...)`` command.
This can be done globally:

.. code-block:: python

   dask.config.set(scheduler='threads')

   x.compute()

or as a context manager:

.. code-block:: python

   with dask.config.set(scheduler='threads'):
       x.compute()

or within a single compute call:

.. code-block:: python

   x.compute(scheduler='threads')

Each scheduler may support extra keywords specific to that scheduler. For example,
the pool-based single-machine scheduler allows you to provide custom pools or
specify the desired number of workers:

.. code-block:: python

   from concurrent.futures import ThreadPoolExecutor
   with dask.config.set(pool=ThreadPoolExecutor(4)):
       x.compute()

   with dask.config.set(num_workers=4):
       x.compute()

Note that Dask also supports custom ``concurrent.futures.Executor`` subclasses,
such as the ``ReusablePoolExecutor`` from loky_:

.. _loky: https://github.com/joblib/loky

.. code-block:: python

   from loky import get_reusable_executor
   with dask.config.set(scheduler=get_reusable_executor()):
       x.compute()

Other libraries like ipyparallel_ and mpi4py_ also supply
``concurrent.futures.Executor`` subclasses that could be used as well.

.. _ipyparallel: https://ipyparallel.readthedocs.io/en/latest/examples/Futures.html#Executors
.. _mpi4py: https://mpi4py.readthedocs.io/en/latest/mpi4py.futures.html


Standalone Python scripts
-------------------------

Some care needs to be taken when running Dask schedulers in a standalone Python script.
Specifically, when using the single-machine multiprocessing scheduler or the local distributed
scheduler, Dask will create additional Python processes. As part of Python's normal subprocess
initialization, Python will import the contents of the script in every child process that is created
(this is true for any Python code where child processes are created -- not just in Dask).
This import initialization can lead to subprocesses recursively creating other subprocesses
and eventually an error is raised.

.. dropdown:: Common error encountered

   .. code-block:: python

      An attempt has been made to start a new process before the
      current process has finished its bootstrapping phase.

      This probably means that you are not using fork to start your
      child processes and you have forgotten to use the proper idiom
      in the main module:

         if __name__ == '__main__':
               freeze_support()
               ...

      The "freeze_support()" line can be omitted if the program
      is not going to be frozen to produce an executable.

To avoid this types of error, you should place any Dask code that create subprocesses
(for example, all ``compute()`` calls that use the multiprocessing scheduler, or when creating
a local distributed cluster) inside a ``if __name__ == "__main__":`` block. This ensures
subprocesses are only created when your script is run as the main program.

For example, running ``python myscript.py`` with the script below will raise an error:

.. code-block:: python

   # myscript.py

   from dask.distributed import Client
   client = Client()  # Will raise an error when creating local subprocesses


Instead one should place the contents of the script inside a ``if __name__ == "__main__":`` block:

.. code-block:: python

   # myscript.py

   if __name__ == "__main__":  # This avoids infinite subprocess creation

      from dask.distributed import Client
      client = Client()
       
For more details on this topic see
`Python's multiprocessing guidelines <https://docs.python.org/3/library/multiprocessing.html#programming-guidelines>`_.
