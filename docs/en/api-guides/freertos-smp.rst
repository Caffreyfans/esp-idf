ESP-IDF FreeRTOS SMP Changes
============================

Overview
--------

.. only:: not CONFIG_FREERTOS_UNICORE

  The vanilla FreeRTOS is designed to run on a single core. However the {IDF_TARGET_NAME} is
  dual core containing a Protocol CPU (known as **CPU 0** or **PRO_CPU**) and an
  Application CPU (known as **CPU 1** or **APP_CPU**). The two cores are
  identical in practice and share the same memory. This allows the two cores to
  run tasks interchangeably between them.

The ESP-IDF FreeRTOS is a modified version of vanilla FreeRTOS which supports
symmetric multiprocessing (SMP). ESP-IDF FreeRTOS is based on the Xtensa port
of FreeRTOS v10.2.0. This guide outlines the major differences between vanilla
FreeRTOS and ESP-IDF FreeRTOS. The API reference for vanilla FreeRTOS can be
found via https://www.freertos.org/a00106.html

For information regarding features that are exclusive to ESP-IDF FreeRTOS,
see :doc:`ESP-IDF FreeRTOS Additions</api-reference/system/freertos_additions>`.

.. only:: not CONFIG_FREERTOS_UNICORE

  :ref:`tasks-and-task-creation`: Use :cpp:func:`xTaskCreatePinnedToCore` or 
  :cpp:func:`xTaskCreateStaticPinnedToCore` to create tasks in ESP-IDF FreeRTOS. The 
  last parameter of the two functions is ``xCoreID``. This parameter specifies 
  which core the task is pinned to. Acceptable values are ``0`` for **PRO_CPU**, 
  ``1`` for **APP_CPU**, or ``tskNO_AFFINITY`` which allows the task to run on
  both.

  :ref:`round-robin-scheduling`: The ESP-IDF FreeRTOS scheduler implements a "Best Effort Round-Robin Scheduling" instead of the ideal Round-Robin scheduling in vanilla FreeRTOS.

  :ref:`scheduler-suspension`: Suspending the scheduler in ESP-IDF FreeRTOS will only 
  affect the scheduler on the the calling core. In other words, calling 
  :cpp:func:`vTaskSuspendAll` on **PRO_CPU** will not prevent **APP_CPU** from scheduling, and
  vice versa. Use critical sections or semaphores instead for simultaneous
  access protection.

  :ref:`tick-interrupt-synchronicity`: Tick interrupts of **PRO_CPU** and **APP_CPU** 
  are not synchronized. Do not expect to use :cpp:func:`vTaskDelay` or 
  :cpp:func:`vTaskDelayUntil` as an accurate method of synchronizing task execution 
  between the two cores. Use a counting semaphore instead as their context 
  switches are not tied to tick interrupts due to preemption.

  :ref:`critical-sections`: In ESP-IDF FreeRTOS, critical sections are implemented using
  mutexes. Entering critical sections involve taking a mutex, then disabling the 
  scheduler and interrupts of the calling core. However the other core is left 
  unaffected. If the other core attemps to take same mutex, it will spin until
  the calling core has released the mutex by exiting the critical section.

.. only:: esp32

  :ref:`floating-points`: The ESP32 supports hardware acceleration of single
  precision floating point arithmetic (``float``). However the use of hardware
  acceleration leads to some behavioral restrictions in ESP-IDF FreeRTOS.
  Therefore, tasks that utilize ``float`` will automatically be pinned to a core if 
  not done so already. Furthermore, ``float`` cannot be used in interrupt service 
  routines.

:ref:`deletion-callbacks`: Deletion callbacks are called automatically during task deletion and are
used to free memory pointed to by TLSP. Call 
:cpp:func:`vTaskSetThreadLocalStoragePointerAndDelCallback()` to set TLSP and Deletion
Callbacks.

:ref:`esp-idf-freertos-configuration`: Several aspects of ESP-IDF FreeRTOS can be
set in the project configuration (``idf.py menuconfig``) such as running ESP-IDF in
Unicore (single core) Mode, or configuring the number of Thread Local Storage Pointers
each task will have.

It is not necessary to manually start the FreeRTOS scheduler by calling :cpp:func:`vTaskStartScheduler`. In ESP-IDF the
scheduler is started by the :doc:`startup` and is already running when the ``app_main`` function is called (see :ref:`app-main-task` for details).

.. _tasks-and-task-creation:

Tasks and Task Creation
-----------------------

Tasks in ESP-IDF FreeRTOS are designed to run on a particular core, therefore
two new task creation functions have been added to ESP-IDF FreeRTOS by
appending ``PinnedToCore`` to the names of the task creation functions in
vanilla FreeRTOS. The vanilla FreeRTOS functions of :cpp:func:`xTaskCreate`
and :cpp:func:`xTaskCreateStatic` have led to the addition of 
:cpp:func:`xTaskCreatePinnedToCore` and :cpp:func:`xTaskCreateStaticPinnedToCore` in 
ESP-IDF FreeRTOS 

For more details see :component_file:`freertos/FreeRTOS-Kernel/tasks.c`

The ESP-IDF FreeRTOS task creation functions are nearly identical to their
vanilla counterparts with the exception of the extra parameter known as
``xCoreID``. This parameter specifies the core on which the task should run on
and can be one of the following values.

    -	``0`` pins the task to **PRO_CPU**
    -	``1`` pins the task to **APP_CPU**
    -	``tskNO_AFFINITY`` allows the task to be run on both CPUs

For example ``xTaskCreatePinnedToCore(tsk_callback, “APP_CPU Task”, 1000, NULL, 10, NULL, 1)``
creates a task of priority 10 that is pinned to **APP_CPU** with a stack size
of 1000 bytes. It should be noted that the ``uxStackDepth`` parameter in
vanilla FreeRTOS specifies a task’s stack depth in terms of the number of
words, whereas ESP-IDF FreeRTOS specifies the stack depth in terms of bytes.

Note that the vanilla FreeRTOS functions :cpp:func:`xTaskCreate` and
:cpp:func:`xTaskCreateStatic` have been defined in ESP-IDF FreeRTOS as inline functions which call
:cpp:func:`xTaskCreatePinnedToCore` and :cpp:func:`xTaskCreateStaticPinnedToCore`
respectively with ``tskNO_AFFINITY`` as the ``xCoreID`` value.

Each Task Control Block (TCB) in ESP-IDF stores the ``xCoreID`` as a member.
Hence when each core calls the scheduler to select a task to run, the
``xCoreID`` member will allow the scheduler to determine if a given task is
permitted to run on the core that called it.

Scheduling
----------

The vanilla FreeRTOS implements scheduling in the ``vTaskSwitchContext()``
function. This function is responsible for selecting the highest priority task
to run from a list of tasks in the Ready state known as the Ready Tasks List
(described in the next section). In ESP-IDF FreeRTOS, each core will call
``vTaskSwitchContext()`` independently to select a task to run from the
Ready Tasks List which is shared between both cores. There are several
differences in scheduling behavior between vanilla and ESP-IDF FreeRTOS such as
differences in Round Robin scheduling, scheduler suspension, and tick interrupt
synchronicity.

.. _round-robin-scheduling:

Round Robin Scheduling
^^^^^^^^^^^^^^^^^^^^^^

Given multiple tasks in the Ready state and of the same priority, vanilla FreeRTOS implements Round Robin scheduling between multiple ready state tasks of the same priority. This will result in running those tasks in turn each time the scheduler is called (e.g. when the tick interrupt occurs or when a task blocks/yields).

On the other hand, it is not possible for the ESP-IDF FreeRTOS scheduler to implement perfect Round Robin due to the fact that a particular task may not be able to run on a particular core due to the following reasons:

- The task is pinned to the another core.
- For unpinned tasks, the task is already being run by another core.

Therefore, when a core searches the ready state task list for a task to run, the core may need to skip over a few tasks in the same priority list or drop to a lower priority in order to find a ready state task that the core can run.

The ESP-IDF FreeRTOS scheduler implements a Best Effort Round Robin scheduling for ready state tasks of the same priority by ensuring that tasks that have been selected to run will be placed at the back of the list, thus giving unselected tasks a higher priority on the next scheduling iteration (i.e., the next tick interrupt or yield)

The following example demonstrates the Best Effort Round Robin Scheduling in action. Assume that:

- There are four ready state tasks of the same priority ``AX, B0, C1, D1`` where:
  - The priority is the current highest priority with ready state tasks
  - The first character represents the task's names (i.e., ``A, B, C, D``)
  - And the second character represents the tasks core pinning (and ``X`` means unpinned)
- The task list is always searched from the head

.. code-block:: none

    --------------------------------------------------------------------------------

    1. Starting state. None of the ready state tasks have been selected to run

    Head [ AX , B0 , C1 , D0 ] Tail

    --------------------------------------------------------------------------------

    2. Core 0 has tick interrupt and searches for a task to run.
      Task A is selected and is moved to the back of the list

    Core0--|
    Head [ AX , B0 , C1 , D0 ] Tail

                          0
    Head [ B0 , C1 , D0 , AX ] Tail

    --------------------------------------------------------------------------------

    3. Core 1 has a tick interrupt and searches for a task to run.
      Task B cannot be run due to incompatible affinity, so core 1 skips to Task C.
      Task C is selected and is moved to the back of the list

    Core1-------|         0
    Head [ B0 , C1 , D0 , AX ] Tail

                     0    1
    Head [ B0 , D0 , AX , C1 ] Tail

    --------------------------------------------------------------------------------

    4. Core 0 has another tick interrupt and searches for a task to run.
      Task B is selected and moved to the back of the list


    Core0--|              1
    Head [ B0 , D0 , AX , C1 ] Tail

                     1    0
    Head [ D0 , AX , C1 , B0 ] Tail

    --------------------------------------------------------------------------------

    5. Core 1 has another tick and searches for a task to run.
      Task D cannot be run due to incompatible affinity, so core 1 skips to Task A
      Task A is selected and moved to the back of the list

    Core1-------|         0
    Head [ D0 , AX , C1 , B0 ] Tail

                     0    1
    Head [ D0 , C1 , B0 , AX ] Tail


The implications to users regarding the Best Effort Round Robin Scheduling:

- Users cannot expect multiple ready state tasks of the same priority to run sequentially (as is the case in Vanilla FreeRTOS). As demonstrated in the example above, a core may need to skip over tasks.
- However, given enough ticks, a task will eventually be given some processing time.
- If a core cannot find a task runnable task at the highest ready state priority, it will drop to a lower priority to search for tasks.
- To achieve ideal round robin scheduling, users should ensure that all tasks of a particular priority are pinned to the same core.


.. _scheduler-suspension:

Scheduler Suspension
^^^^^^^^^^^^^^^^^^^^

In vanilla FreeRTOS, suspending the scheduler via :cpp:func:`vTaskSuspendAll` will
prevent calls of ``vTaskSwitchContext`` from context switching until the
scheduler has been resumed with :cpp:func:`xTaskResumeAll`. However servicing ISRs
are still permitted. Therefore any changes in task states as a result from the
current running task or ISRs will not be executed until the scheduler is
resumed. Scheduler suspension in vanilla FreeRTOS is a common protection method
against simultaneous access of data shared between tasks, whilst still allowing
ISRs to be serviced.

In ESP-IDF FreeRTOS, :cpp:func:`xTaskSuspendAll` will only prevent calls of
``vTaskSwitchContext()`` from switching contexts on the core that called for the
suspension. Hence if **PRO_CPU** calls :cpp:func:`vTaskSuspendAll`, **APP_CPU** will
still be able to switch contexts. If data is shared between tasks that are
pinned to different cores, scheduler suspension is **NOT** a valid method of
protection against simultaneous access. Consider using critical sections
(disables interrupts) or semaphores (does not disable interrupts) instead when
protecting shared resources in ESP-IDF FreeRTOS.

In general, it's better to use other RTOS primitives like mutex semaphores to protect
against data shared between tasks, rather than :cpp:func:`vTaskSuspendAll`.


.. _tick-interrupt-synchronicity:

Tick Interrupt Synchronicity
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

In ESP-IDF FreeRTOS, tasks on different cores that unblock on the same tick
count might not run at exactly the same time due to the scheduler calls from
each core being independent, and the tick interrupts to each core being
unsynchronized.

In vanilla FreeRTOS the tick interrupt triggers a call to
:cpp:func:`xTaskIncrementTick` which is responsible for incrementing the tick
counter, checking if tasks which have called :cpp:func:`vTaskDelay` have fulfilled
their delay period, and moving those tasks from the Delayed Task List to the
Ready Task List. The tick interrupt will then call the scheduler if a context
switch is necessary.

In ESP-IDF FreeRTOS, delayed tasks are unblocked with reference to the tick
interrupt on PRO_CPU as PRO_CPU is responsible for incrementing the shared tick
count. However tick interrupts to each core might not be synchronized (same
frequency but out of phase) hence when PRO_CPU receives a tick interrupt,
APP_CPU might not have received it yet. Therefore if multiple tasks of the same
priority are unblocked on the same tick count, the task pinned to PRO_CPU will
run immediately whereas the task pinned to APP_CPU must wait until APP_CPU
receives its out of sync tick interrupt. Upon receiving the tick interrupt,
APP_CPU will then call for a context switch and finally switches contexts to
the newly unblocked task.

Therefore, task delays should **NOT** be used as a method of synchronization
between tasks in ESP-IDF FreeRTOS. Instead, consider using a counting semaphore
to unblock multiple tasks at the same time.


.. _critical-sections:

Critical Sections & Disabling Interrupts
----------------------------------------

Vanilla FreeRTOS implements critical sections with ``taskENTER_CRITICAL()`` which
calls ``portDISABLE_INTERRUPTS()``. This prevents preemptive context switches and
servicing of ISRs during a critical section. Therefore, critical sections are
used as a valid protection method against simultaneous access in vanilla FreeRTOS.

.. only:: not CONFIG_FREERTOS_UNICORE

    On the other hand, {IDF_TARGET_NAME} has no hardware method for cores to disable each
    other’s interrupts. Calling ``portDISABLE_INTERRUPTS()`` will have no effect on
    the interrupts of the other core. Therefore, disabling interrupts is **NOT**
    a valid protection method against simultaneous access to shared data as it
    leaves the other core free to access the data even if the current core has
    disabled its own interrupts.

.. only:: CONFIG_FREERTOS_UNICORE

   ESP-IDF contains some modifications to work with dual core concurrency,
   and the dual core API is used even on a single core only chip.

For this reason, ESP-IDF FreeRTOS implements critical sections using special
mutexes, referred by ``portMUX_Type`` objects. These are implemented on top of a
specific spinlock component.  Calls to ``taskENTER_CRITICAL`` or
``taskEXIT_CRITICAL`` each provide a spinlock object as an argument. The
spinlock is associated with a shared resource requiring access protection.  When
entering a critical section in ESP-IDF FreeRTOS, the calling core will disable
interrupts similar to the vanilla FreeRTOS implementation, and will then take the
spinlock and enter the critical section. The other core is unaffected at this point,
unless it enters its own critical section and attempts to take the same spinlock.
In that case it will spin until the lock is released. Therefore, the ESP-IDF FreeRTOS
implementation of critical sections allows a core to have protected access to a shared
resource without disabling the other core. The other core will only be affected if it
tries to concurrently access the same resource.

The ESP-IDF FreeRTOS critical section functions have been modified as follows…

 - ``taskENTER_CRITICAL(mux)``, ``taskENTER_CRITICAL_ISR(mux)``,
   ``portENTER_CRITICAL(mux)``, ``portENTER_CRITICAL_ISR(mux)`` are all macro
   defined to call internal function :cpp:func:`vPortEnterCritical`

 - ``taskEXIT_CRITICAL(mux)``, ``taskEXIT_CRITICAL_ISR(mux)``,
   ``portEXIT_CRITICAL(mux)``, ``portEXIT_CRITICAL_ISR(mux)`` are all macro
   defined to call internal function :cpp:func:`vPortExitCritical`

 - ``portENTER_CRITICAL_SAFE(mux)``, ``portEXIT_CRITICAL_SAFE(mux)`` macro identifies
   the context of execution, i.e ISR or Non-ISR, and calls appropriate critical
   section functions (``port*_CRITICAL`` in Non-ISR and ``port*_CRITICAL_ISR`` in ISR)
   in order to be in compliance with Vanilla FreeRTOS.

For more details see :component_file:`esp_hw_support/include/soc/spinlock.h`,
:component_file:`freertos/FreeRTOS-Kernel/include/freertos/task.h`,
and :component_file:`freertos/FreeRTOS-Kernel/tasks.c`

It should be noted that when modifying vanilla FreeRTOS code to be ESP-IDF
FreeRTOS compatible, it is trivial to modify the type of critical section called
as they are all defined to call the same function. As long as the same spinlock
is provided upon entering and exiting, the exact macro or function used for the
call should not matter.


.. only:: not CONFIG_FREERTOS_UNICORE

    .. _floating-points:

    Floating Point Arithmetic
    -------------------------

    ESP-IDF FreeRTOS implements Lazy Context Switching for FPUs. In other words,
    the state of a core's FPU registers are not immediately saved when a context
    switch occurs. Therefore, tasks that utilize ``float`` must be pinned to a
    particular core upon creation. If not, ESP-IDF FreeRTOS will automatically pin
    the task in question to whichever core the task was running on upon the task's
    first use of ``float``. Likewise due to Lazy Context Switching, only interrupt
    service routines of lowest priority (that is it the Level 1) can use ``float``,
    higher priority interrupts do not support FPU usage.

    ESP32 does not support hardware acceleration for double precision floating point
    arithmetic (``double``). Instead ``double`` is implemented via software hence the
    behavioral restrictions with regards to ``float`` do not apply to ``double``. Note
    that due to the lack of hardware acceleration, ``double`` operations may consume
    significantly larger amount of CPU time in comparison to ``float``.

.. _task-deletion:

Task Deletion
-------------

In FreeRTOS task deletion the freeing of task memory will occur
immediately (within :cpp:func:`vTaskDelete`) if the task being deleted is not currently 
running or is not pinned to the other core (with respect to the core 
:cpp:func:`vTaskDelete` is called on). TLSP deletion callbacks will also run immediately
if the same conditions are met.

However, calling :cpp:func:`vTaskDelete` to delete a task that is either currently
running or pinned to the other core will still result in the freeing of memory
being delegated to the Idle Task.


.. _deletion-callbacks:

Thread Local Storage Pointers & Deletion Callbacks
--------------------------------------------------

Thread Local Storage Pointers (TLSP) are pointers stored directly in the TCB.
TLSP allow each task to have its own unique set of pointers to data structures.
However task deletion behavior in vanilla FreeRTOS does not automatically
free the memory pointed to by TLSP. Therefore if the memory pointed to by
TLSP is not explicitly freed by the user before task deletion, memory leak will
occur.

ESP-IDF FreeRTOS provides the added feature of Deletion Callbacks. Deletion
Callbacks are called automatically during task deletion to free memory pointed
to by TLSP. Each TLSP can have its own Deletion Callback. Note that due to the
to `Task Deletion`_ behavior, there can be instances where Deletion
Callbacks are called in the context of the Idle Tasks. Therefore Deletion
Callbacks **should never attempt to block** and critical sections should be kept
as short as possible to minimize priority inversion.

Deletion callbacks are of type
``void (*TlsDeleteCallbackFunction_t)( int, void * )`` where the first parameter
is the index number of the associated TLSP, and the second parameter is the
TLSP itself.

Deletion callbacks are set alongside TLSP by calling
:cpp:func:`vTaskSetThreadLocalStoragePointerAndDelCallback`. Calling the vanilla
FreeRTOS function :cpp:func:`vTaskSetThreadLocalStoragePointer` will simply set the
TLSP's associated Deletion Callback to `NULL` meaning that no callback will be
called for that TLSP during task deletion. If a deletion callback is `NULL`,
users should manually free the memory pointed to by the associated TLSP before
task deletion in order to avoid memory leak.

For more details see :doc:`FreeRTOS API reference<../api-reference/system/freertos>`.


.. _esp-idf-freertos-configuration:

Configuring ESP-IDF FreeRTOS
----------------------------

The ESP-IDF FreeRTOS can be configured in the project configuration menu
(``idf.py menuconfig``) under ``Component Config/FreeRTOS``. The following section
highlights some of the ESP-IDF FreeRTOS configuration options. For a full list of
ESP-IDF FreeRTOS configurations, see :doc:`FreeRTOS <../api-reference/kconfig>`

.. only:: not CONFIG_FREERTOS_UNICORE

    :ref:`CONFIG_FREERTOS_UNICORE` will run ESP-IDF FreeRTOS only
    on **PRO_CPU**. Note that this is **not equivalent to running vanilla
    FreeRTOS**. Note that this option may affect behavior of components other than
    :component:`freertos`. For more details regarding the
    effects of running ESP-IDF FreeRTOS on a single core, search for
    occurences of ``CONFIG_FREERTOS_UNICORE`` in the ESP-IDF components.

.. only:: CONFIG_FREERTOS_UNICORE

    As {IDF_TARGET_NAME} is a single core SoC, the config item :ref:`CONFIG_FREERTOS_UNICORE` is
    always set. This means ESP-IDF only runs on the single CPU. Note that this is **not
    equivalent to running vanilla FreeRTOS**. Behaviors of multiple components in ESP-IDF
    will be modified. For more details regarding the effects of running ESP-IDF FreeRTOS
    on a single core, search for occurences of ``CONFIG_FREERTOS_UNICORE`` in the ESP-IDF components.

:ref:`CONFIG_FREERTOS_ASSERT_ON_UNTESTED_FUNCTION` will trigger a halt in
particular functions in ESP-IDF FreeRTOS which have not been fully tested
in an SMP context.

:ref:`CONFIG_FREERTOS_TASK_FUNCTION_WRAPPER` will enclose all task functions
within a wrapper function. In the case that a task function mistakenly returns
(i.e. does not call :cpp:func:`vTaskDelete`), the call flow will return to the
wrapper function. The wrapper function will then log an error and abort the
application, as illustrated below::

    E (25) FreeRTOS: FreeRTOS task should not return. Aborting now!
    abort() was called at PC 0x40085c53 on core 0
