# Wisconsin Scheduler

## Updates

- global_tickets - this is the sum of tickets of all processes wanting the resource (CPU in our case), this includes the process that is running as well as all processes ready to be run.
- `getpinfo` should return -1 in the case it fails.
- A helpful explanation of scheduling in xv6 https://www.youtube.com/watch?v=eYfeOT1QYmg
- ### Clarification on remain. Why do I need it?

  Think about stride as speed each process is running at, and pass as the total distance the process has covered. The goal of the stride scheduler in our analogy is to keep the processes as close as possible. We don't want any process to be left behind. We always pick the process which is the farthest behind.

  One way to think about it is the scheduler is trying to get it each process to atleast a threshold before letting any process run ahead. This value is the `global_pass`.

  Thinking in terms of speed will also clarify why we calculate `global_pass` as `STRIDE1/global_tickets` instead of taking an average of all `strides`. (Think back to when you needed to calculate the average speed in your physics class, you cannot just take the average of speeds).

  Because each process has a different speed, a process can cover variable amount of distance in one time unit. So a process with speed (stride) 100 will cover 100 units in one tick, while a process with speed (stride) 1, will just cover 1.

  For this reason, a process may be ahead or behind the average of the entire system at the time it goes to sleep. Let us consider 2 processes, `A` with stride 100 and `B` with stride 1, both of them have pass 0 at the start.

  Process `A` is scheduled first. Making its pass 100, now ideally we will schedule `B` 100 times before process `A` is scheduled again. After 20 times, `B` goes to sleep. Now when `B` is runnable again, we need to boost its pass value. We cannot use the same logic as setting it to the system average i.e. `global_pass` as it will be unfair as `B` was behind the average when it went to sleep. It should get more preference after being awake. This advantage/disadvantage compared to the global average is now encoded in terms of remain.

# Project Details

## Overview of Basic Stride Scheduling

The stride scheduler maintains a few additional pieces of information for each process:

- `tickets` -- a value assigned upon process creation. It can be modified by a system call. It should default to 8.
- `stride` -- a value that is inversely proportional to a process' tickets. `stride = STRIDE1 / tickets` where `STRIDE1` is a constant (for this project: 1<<10).
- `pass` -- initially set to `0`. This gets updated every time the process runs.

When the stride scheduler runs, it selects the runnable process with the lowest `pass` value to run for the next tick.
After the process runs, the scheduler increments its `pass` value by its `stride`: `pass += stride`. These steps ensures that over time,
each process receives CPU time proportional to its tickets.

## Dynamic Process Participation

The Basic Stride Algorithm does not account for changes to the total number of processes waiting to be scheduled.

Consider the case when we have 2 long running processes which have already been running and they all currently have a `stride` value of 1 and `pass` value of 100.
A new process, let us say `A` now joins our system with `stride` value of 1.

What happens in the case of Basic Stride Scheduling?

Because the `pass` value of `A` is so small compared to the other processes, it will now be scheduled for the next 100 ticks before any other process is allowed to run.
This is not the behavior we want. Given each process has equal tickets, we want the CPU to be shared equally among all of the processes including the newly arrived process.
In this particular case we would want all processes to take turns.

#### How do we do that?

Let us maintain aggregate information about the set of processes waiting to be scheduled and use that information when a process enters or leaves.

- `global_tickets` -- the sum of all **runnable** process's tickets.
- `global_stride` -- inversely proportional to the `global_tickets`, specifically `STRIDE1 / global_tickets`
- `global_pass` -- incremented by the **current** `global_stride` at every tick.

Now, when a process is created, its `pass` value will begin at `global_pass`. In the case of process `A` above, the `global_stride` and number of ticks that have occurred
will make `A`'s starting `pass` value be the same as the other 2 processes.

The global variables will need to be recalculated whenever a process enters or leaves to create the intended behaviour.

The final piece of information the scheduler will need to keep track for each process is the `remain` value which will store the remaining portion
of its stride when a dynamic change occurs. The `remain` field represents the number of passes that are left before a process' next selection.
When a process leaves the scheduler queue, `remain` is computed as the difference between the process' `pass` and the `global_pass`.
Then when a process rejoins the system, its `pass` value is recomputed by adding its `remain` value to the `global_pass`.

This mechanism handles situations involving either positive or negative error between the specified and actual number of allocations.

- If remain < stride, then the process is effectively given credit when it rejoins for having previously waited for part of its stride without receiving a timeslice tick to run.
- If remain > stride, then the process is effectively penalized when it rejoins for having previously received a timeslice tick without waiting for its entire stride.

Let us consider an example, process `A` currently has a pass of 1000, where the `global_pass` is 600. Now process `A` decides to sleep on keyboard inturrupt. After a few ticks, when the interrupt occurs, the `global_pass` has updated to a 1400. We only want to increment `A`'s pass for the time it was asleep, so we cannot just add 1400 to 1000. Instead we measure `remain` at the time process left the scheduler queue, in this case 1000-600=400, and when process `A` rejoins we will calculate the new pass as `remain+global_pass` that is 400+1400= 1800.

## Dynamic Ticket Modification

We also want to support dynamically changing a process’ ticket allocation.

When a process’ allocation is dynamically changed from `tickets`to `tickets'`, its stride and pass values must be recomputed. The new `stride'` is computed as usual, inversely
proportional to tickets.

To compute the new `pass'`, the remaining portion of the client’s current stride, denoted by `remain`, is adjusted to reflect the new `stride'`. This is accomplished

by scaling `remain` by `stride'/stride`
