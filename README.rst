Operating System From Scratch
-----------------------------

Step 10: Memory Management
``````````````````````````

You might do *fork*\ s a lot in GitHub.
If so, it'll be easy for you to understand *fork* in the OS world.
Today we will implement a *fork* in our own OS.

*fork* has some friends, they are *exec*, *wait* and *exit*.
We'll implement them too.

fork
''''

So far all the processes are hard coded.
Doing *fork* implies we will create/destroy processes dynamically.
Before starting, let's recall the data structures related to a process::

        process-related stuff:
                                                                                                           
              16         0
            H +----------+
              :          :
              +----------+
              |ldt desc  |        16            0
              |for proc j|      H +-------------+          / H +-----+
              +----------+        :             :         |    |Stack|
              :          :      / +-------------+         |    +-----+
              +----------+     |  |   ldts[RW]  |---.     |    :     :
              | ldt desc +--->-|  +-------------+   +--->-|    +-----+
         .--> |for proc i|  X  |  |   ldts[C]   |---` Z   |    |Data |
         |    +----------+      \ +-------------+         |    +-----+
         |    :          :        |   ldt_sel   +--.      |    |Text |
         |  L +----------+        +-------------+  |       \ L +-----+
         |         GDT            : STACK FRAME :  |        process image
         |                      L +-------------+  |
         |                         proc_table[i]   |
         |                                         |
         `-----------------------------------------`
                             Y

There are three data structures: ``GDT``, ``proc_table[]`` and the process image.
There are three relations between them: X, Y and Z.

In the past, all of them are prepared before the processes run:

+ ``GDT`` is prepared in ``kernel/main.c``
+ ``proc_table[]`` is prepared in ``kernel/global.c``
+ process images are the compiled binaries of the process functions (e.g. ``task_fs()`` for FS)
+ relations X is prepared in ``init_proc()``
+ relations Y and Z are prepared in ``kernel_main()``

We can see that although we will create processes dynamically,
some of the above works can still be prepared beforehand:

+ We can reserve slots in ``proc_table[]`` and ``GDT`` for new processes
+ We can set all ``ldt_sel``\ s in ``proc_table[]`` properly (relation Y)
+ We can set all ldt descriptors in ``GDT`` properly (relation X)

The memory allocation for the process image and the relation Z must be done when doing *fork*.

See ``a/`` for more details.

exit and wait
'''''''''''''

If process A calls ``exit()``, then MM will do the following:

1. inform FS so that the fd-related things will be cleaned up
2. free A's memory
3. set A\ ``.exit_status``, which is for the parent
4. depends on parent's status. if parent (say P) is:

   1. ``WAITING``

      + clean P's ``WAITING`` bit, and
      + send P a message to unblock it
      + release A's ``proc_table[]`` slot

   2. not ``WAITING``

      + set A's ``HANGING`` bit

5. iterate ``proc_table[]``, if proc B is found as A's child, then:

   1. make INIT the new parent of B, and

   2. if INIT is ``WAITING`` and B is ``HANGING``, then:

      + clean INIT's ``WAITING`` bit, and
      + send INIT a message to unblock it
      + release B's ``proc_table[]`` slot

If process P calls ``wait()``, then MM will do the following in this routine:

1. iterate ``proc_table[]``, if proc A is found as P's child and it is ``HANGING``

   - reply to P
   - release A's ``proc_table[]`` entry

2. if no child of P is ``HANGING``

   - set P's ``WAITING`` bit

3. if P has no child at all

   - reply to P with error

TERMs:
    - ``HANGING``: everything except the ``proc_table`` entry has been cleaned up.
    - ``WAITING``: a process has at least one child, and it is waiting for the child(ren) to ``exit()``

See ``b/`` for details.

exec
''''

Once we implement ``exec()``, we can write programs for our OS.
The figure below is the result of a ``fork()`` followed by ``execl("/echo", "echo", "hello", "world", 0)``.
Calling ``execl("/echo", "echo", "hello", "world", 0)`` is just the same as execute ``echo hello world`` in a shell, except we have no shell yet.

.. image:: http://osfromscratch.org/snapshots/original/%E5%9B%BE10.06%20echo.png

We did *fork* and *exec* successfully and the "``hello world``" appears!

See ``c/`` for more.

shell
'''''

We didn't have a shell when we called ``execl()``, but we will soon.
Armed with ``fork()`` and ``exec()``, it's easy to write a simple shell.
Let's call it *shabby_shell*.

Look, *shabby_shell* is running:

.. image:: http://osfromscratch.org/snapshots/original/%E5%9B%BE10.08%20Shabby%20Shell.png

`‹prev`_   `next›`_

.. _`‹prev`: https://github.com/yyu/osfs09
.. _`next›`: https://github.com/yyu/osfs11
