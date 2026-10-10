# Programming Assignment 2 Type 2
Shortest Remaining Time First (SRTF) Scheduler<br>
Authored by: Alyssa Rahma Adjani & Paramita Santoso<br>

## Member Contributions
| Member | Student ID | Contribution |
|---|---|---|
| Alyssa Rahma Adjani | 2506558466 | - Developed the preemptive Shortest Remaining Time First (SRTF) decision logic in `choose_next()` in `scheduler.c`<br>- Implemented preemption detection and event logging in `choose_next()`, and event reporting in `print_topic_extra()`<br>- Ran and verified test cases<br>- Prepared the presentation slides in Google Slides |
| Paramita Santoso | 2506554171 | - Completed part 2 of `fork_vs_thread.c`<br>- Ran and verified test cases, captured stdout logs, and verified CSV outputs<br>- Authored `explanation.md` |
<br>

## Compilation & Execution
Project structure:<pre>
os-pa2-type2/
├── tests/
&nbsp;&nbsp;&nbsp;&nbsp;├── tc1.txt
&nbsp;&nbsp;&nbsp;&nbsp;├── tc2.txt
&nbsp;&nbsp;&nbsp;&nbsp;├── tc3.txt
&nbsp;&nbsp;&nbsp;&nbsp;├── tc4.txt
&nbsp;&nbsp;&nbsp;&nbsp;├── tc5.txt
├── explanation.md
├── fork_vs_thread.c
├── Makefile
├── pcb.h
├── scheduler.c
├── simlib.c
└── simlib.h
</pre>
<br>
How to run:
```bash
# Clean previous build and compile executables
make clean && make

# Execute scheduler for all test cases
./scheduler tests/tc1.txt
./scheduler tests/tc2.txt
./scheduler tests/tc3.txt
./scheduler tests/tc4.txt
./scheduler tests/tc5.txt

# Execute verbose mode to observe state transitions
./scheduler tests/tc1.txt -v

# Execute thread vs process memory isolation experiment
./fork_vs_thread

# Check memory leaks with Valgrind
valgrind ./scheduler tests/tc1.txt
```
<br>

## How the Algorithm Decides
### SRTF Decision Logic (choose_next)
Shortest Remaining Time First (SRTF) is the preemptive variant of Shortest Job First (SJF). At every discrete simulation time unit t:
1. The scheduler scans all processes in sim->proc[] that have arrived (`p->arrival <= sim->clock`) and are unfinished (`p->remaining > 0`).
2. Selects the candidate process with the smallest remaining CPU time (p->remaining).
3. **Tie-Breaking Rule**: If two or more candidate processes have identical remaining times, ties are broken strictly by selecting the candidate with the lowest PID.
4. **Preemption**: Unlike FCFS, the choice is re-evaluated every single tick. If a newly arrived process requires less CPU time than the currently running process has left, it preempts the running process immediately. A process with equal remaining time also takes over if it has a lower PID.
<br>

## Why the Simulation is Repeatable
Although every process is instantiated as an active POSIX thread (pthread_t), execution is 100% deterministic across runs due to a strict semaphore handshake:
* Each process thread blocks on its private semaphore (`p->run_sem`)
* At time tick t, the scheduler posts `p->run_sem` for the selected process and blocks on `sim->tick_done`.
* The selected process thread wakes up, decrements `p->remaining` by 1 unit, posts `sim->tick_done` back to the scheduler, and blocks again.
* The only decision-making code, choose_next(), always breaks ties by lowest PID.

Because the scheduler grants one tick to a selected process and waits for it to finish, only one simulated process executes a tick at a time. The semaphore handshake and lowest-PID tie rule keep the simulated schedule repeatable across runs. <br>

## What `fork_vs_thread.c` Showed
In virtual machine:
```
=== Part 1: two THREADS sharing one counter ===
  expected : 200000
  actual   : 200000
  -> no race observed this run (try again; a race is not
     guaranteed to show itself every time, which is exactly
     what makes race conditions hard to find)

=== Part 2: a forked PROCESS with its own copy ===
[Child] Counter value: 999
[Parent] Counter value: 0
```
<br>

On the laptop terminal:
```
=== Part 1: two THREADS sharing one counter ===
  expected : 200000
  actual   : 174822
  -> the counter is WRONG: the two threads raced on counter++

=== Part 2: a forked PROCESS with its own copy ===
[Child] Counter value: 999
[Parent] Counter value: 0
```

* **Part 1 (Threads)**: Threads share the same virtual address space. Both threads update `counter` without synchronization, creating a data race that can cause lost updates. The laptop run showed an incorrect result, while the VM run happened to produce 200000. A correct-looking result does not prove the absence of a race; thread scheduling and compiler optimizations can affect whether its effects are visible.
* **Part 2 (Processes / fork)**: Calling fork() creates a child process with a separate Copy-On-Write address space. Modifying counter = 999 in the child process affects only the child's memory. When the parent resumes after `wait(NULL)`, its counter remains 0.

### Why do the scheduler threads need semaphores, while the separate counter copies do not?
Because threads share memory, all process threads in `scheduler.c` share access to the same `Sim` struct and `PCB` records. `Sim` is a static local variable in `main()`, shared with the threads through pointers. The semaphores coordinate each process thread with the scheduler so that one simulated tick finishes before the next decision. In the `fork()` demonstration, the parent and child have separate counter copies, so their counter updates do not need synchronization. Separate processes can still need synchronization if they deliberately share memory.

## Performance Metrics & FCFS Baseline Comparison
Below is the metric summary collected across all five test cases:
The comparison columns show **FCFS → SRTF**. These values come from the captured runs in `sample_output.txt`.

| Test case | Average WT | Average TAT | Average RT | Context switches |
|---|---|---|---|---|
| tc1.txt | 8.60 → 5.20 | 13.00 → 9.60 | 8.60 → 2.00 | 4 → 6 |
| tc2.txt | 1.20 → 0.60 | 4.20 → 3.60 | 1.20 → 0.40 | 4 → 5 |
| tc3.txt | 19.40 → 5.40 | 27.60 → 13.60 | 19.40 → 1.00 | 4 → 6 |
| tc4.txt | 15.70 → 9.00 | 19.80 → 13.10 | 15.70 → 8.20 | 9 → 11 |
| tc5.txt | 7.40 → 5.60 | 11.80 → 10.00 | 7.40 → 5.60 | 4 → 4 |

Additional SRTF results (throughput is in processes per time unit):

| Test case | Preemptions | CPU utilization | Throughput | Total simulation time |
|---|---|---|---|---|
| tc1.txt | 2 | 100.00% | 0.23 | 22 |
| tc2.txt | 1 | 75.00% | 0.25 | 20 |
| tc3.txt | 2 | 100.00% | 0.12 | 41 |
| tc4.txt | 2 | 100.00% | 0.24 | 41 |
| tc5.txt | 0 | 100.00% | 0.23 | 22 |

### What does SRTF do better than FCFS, and why?
SRTF significantly reduces Average Waiting Time (WT), Turnaround Time (TAT), and Response Time (RT).
* Mechanism: FCFS suffers from the Convoy Effect, where short processes are stuck behind long processes. SRTF continuously prioritizes processes with the shortest remaining burst time, allowing short tasks to complete quickly and exit the system.
* Evidence: On `tc3.txt`, SRTF reduces Average Waiting Time from 19.40 to 5.40 (a 72.2% drop) and Average Response Time from 19.40 to 1.00 (a 94.8% drop).<br>

### What does SRTF do worse, or what does it cost?
* More Context Switches: Preempting processes increases the total context switch count. In `tc3.txt`, SRTF incurs 6 context switches versus 4 for FCFS (+50.0%). While this simulator treats context switches as free, real hardware incurs overhead from saving/restoring CPU registers and potentially disrupting cache locality.
* Risk of Starvation: Long processes can be postponed indefinitely if short processes arrive continuously. In `tc3.txt`, P1 (BT=20) arrives at t=0, gets preempted at t=1, and waits until it resumes at t=22, suffering a waiting time of 21 units (22 - 1). P1 eventually finishes, so this test demonstrates a delay rather than actual starvation.
* Burst Time Knowledge: SRTF requires knowing future CPU burst times in advance, which is impractical in general-purpose OS environments.

### On which test case is the difference largest, and why?
The reduction in average waiting time is largest on `tc3.txt`: 19.40 - 5.40 = 14.00 time units (72.2%).
* Reason: `tc3.txt` has uneven burst times (20, 2, 3, 15, 1) with staggered early arrivals. In FCFS, P1 (BT=20) blocks the CPU for 20 units while short processes wait. SRTF preempts P1 at t=1 as soon as P2 (BT=2) arrives, giving short tasks earlier access to the CPU.

### Why preemptions and context switches differ?
* Preemption event: Occurs when a running process that still has work left (`p->remaining > 0`) is forcibly evicted from the CPU because another eligible process is selected by shortest remaining time, with ties broken by lowest PID.
* Context switch: Occurs whenever the CPU hands execution to a different process PID. This includes preemptions and handoffs after completion, including handoffs separated by idle time. The simulator compares the selected PID with the previous busy PID, so the initial dispatch is not counted and resuming from idle counts only if the PID changes.
> In tc3: 6 context switches = 2 preemptions + 4 handoffs after completion.

## Manual Verification: tc5.txt
Manual verification is shown in manual-verification-tc5.pdf 