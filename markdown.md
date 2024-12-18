class: center, middle

# Simple Voting Service

???
Hello everyone! Today, we’ll explore a real-world concurrency problem through a simple Voting Service. 
Understanding concurrency early is crucial because race conditions and thread safety issues can sneak into production code unexpectedly.
---
<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Voting on Different Browsers</h2>
<img src="https://github.com/user-attachments/assets/57972ea6-3e84-4d3e-8612-5da9a1006418" width="700">
</div>
</div>
### https://github.com/TomSpencerLondon/websockets
???
We have a web service where users vote for proposals. For example:

Users submit votes.
The server increments the voteCount.
We expect the final count to reflect all votes accurately, but something goes wrong.
Here’s what users see across browsers. The vote count seems inconsistent. Why is this happening?
---

# The Code Example

```java
public class Proposal {
    private static final Logger logger = LoggerFactory.getLogger(Proposal.class);
    
    private int voteCount;

    public void incrementVoteCount() {
        int before = voteCount;
        voteCount++;
        int after = voteCount;

        logger.info("Before count: " + before + " After count: " + after);
    }
}
```
???
The incrementVoteCount() method looks simple.
But what happens if two threads execute this at the same time?
---
class: center, middle

# The Problem

???
When multiple threads access voteCount simultaneously:

Both threads read the same value.
They increment separately.
One increment overwrites the other.
Result: Lost votes!
---
# Primitive Bytecode
```byte
    GETFIELD com/example/demo/Proposal.voteCount : I
    ICONST_1
    IADD
    PUTFIELD com/example/demo/Proposal.voteCount : I
```
???
The increment operation consists of **three separate steps**: get, add, and put. 
This is problematic because two CPU cores can execute these operations **in parallel**. 
For example, **CPU 1** might perform the "add" operation but not complete the "put," 
while **CPU 2** simultaneously executes the "get" operation on the same memory address. 
As a result, one update can overwrite the other, leading to inconsistencies.

---

# Concurrent Test reveals the issue

```java
public class ProposalServiceTest {
    @Test
    public void testConcurrentVoting() throws InterruptedException {
        Proposal proposal = proposalService.proposals().get(0);
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        int numberOfVotes = 1000;
        Vote vote = new Vote();
        vote.setProposalId(proposal.getId());

        for (int i = 0; i < numberOfVotes; i++) {
            executorService.submit(() -> {
                proposalService.addVote(vote);
            });
        }
        executorService.shutdown();
        executorService.awaitTermination(10, TimeUnit.SECONDS);

        assertEquals(numberOfVotes, proposal.getVoteCount(),
                "Votes should be " + numberOfVotes + " after concurrent voting");
    }
}
```

???
Our test confirms the issue: Expected 1000 votes, but only 985 are recorded. 
Duplicate logs also reveal suspicious counts, where two threads update the same value simultaneously.
---

# Failing Test
```bash
org.opentest4j.AssertionFailedError: Votes should be 1000 
after concurrent voting
Expected :1000
Actual   :985
```
---

# Thread output
```bash
Suspicious duplicate After count: 80
 - 11:47:09.730 [pool-1-thread-89] INFO com.example.demo.Proposal -- Before count: 79 After count: 80
 - 11:47:09.730 [pool-1-thread-74] INFO com.example.demo.Proposal -- Before count: 79 After count: 80
Suspicious duplicate After count: 213
 - 11:47:10.002 [pool-1-thread-60] INFO com.example.demo.Proposal -- Before count: 212 After count: 213
 - 11:47:10.002 [pool-1-thread-9] INFO com.example.demo.Proposal -- Before count: 212 After count: 213
```
???
To detect race conditions, we **parse each log line** to extract the "Before count: X" and "After count: Y" values. 
We then **keep track** of all the "After" values along with the lines where they occurred. 
If the **same "After" value** appears more than once, we print those lines as **suspicious** to highlight potential issues.
---

# Making it Thread-Safe

```java
public synchronized void incrementVoteCount() {
    int before = voteCount;
    voteCount++;
    int after = voteCount;

    logger.info("Before count: " + before + " After count: " + after);
}
```
???
How It Works
"synchronized ensures only one thread can access incrementVoteCount() at a time."
---

<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Scheduler</h2>
<img alt="scheduler" src="https://github.com/user-attachments/assets/d58cef43-a980-4472-8100-b5ba6d0fc0c9" width="500">

</div>
</div>
???
Let’s talk about how threads are managed in a program.

At the top, we have a Java process. A process is a running instance of a program, and within it, we have threads. 
Threads are lightweight units of execution that share the same memory.

When multiple threads are running, the OS Thread Scheduler decides which thread gets access to the CPU at any given moment.

Shared Memory: All threads within the process can access the same variables and data structures.
Thread Execution: The OS scheduler maps these threads to different CPU cores. For example, Thread 1 may execute on CPU 1, while Thread 2 executes on CPU 2.
The scheduler plays a crucial role in ensuring fairness and efficiency when multiple threads compete for CPU time. 
This is why thread management and scheduling are so important for performance and avoiding conflicts.
---

<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Thread States</h2>
<img alt="thread-states" src="https://github.com/user-attachments/assets/ec059494-7e30-4729-bfcd-cba0d9256850" width="900">

</div>
</div>
???
Now, let’s look at the lifecycle of a thread. A thread can be in one of six states:

NEW: The thread is created but hasn’t started running yet.
RUNNABLE: The thread is ready to run or actively executing on the CPU.
BLOCKED: The thread is waiting for a monitor lock to access a resource. This usually happens when another thread holds the lock.
WAITING: The thread is waiting indefinitely for another thread to notify it. For example, it might be waiting for a signal.
TIMED_WAITING: Similar to WAITING, but with a time limit. For instance, a thread is sleeping for a specific period using methods like sleep() or wait(timeout).
TERMINATED: The thread has finished executing or has been stopped.
Understanding thread states helps us debug and manage thread behavior. For example, if a thread is stuck in a BLOCKED or WAITING state, it might indicate a resource contention problem.
---
<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Compare and Swap</h2>
<img alt="compare-and-swap" src="https://github.com/user-attachments/assets/e15433c7-4bff-4854-a97e-f626658f2908" width="500">

</div>
</div>

???
The **Compare-And-Swap (CAS)** mechanism ensures **thread safety** by performing atomic updates without using explicit locks.

Here’s how it works:
1. CAS compares the **current value** of a variable to an **expected value**.
2. If the two values match, the variable is **swapped** with a **new value**.
3. If they don’t match, the operation fails, and the old value is retained.

In Java, `AtomicInteger` uses CAS to implement methods like `incrementAndGet()`. 
This allows the increment operation to be performed **atomically**, even in a multi-threaded environment, without locks.
The advantage of CAS is that it avoids the overhead of locks while ensuring safe, non-blocking updates. 
This makes it ideal for high-performance, thread-safe operations in concurrent systems.

---
<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Compare and Swap</h2>
<img alt="compare-and-swap" src="https://github.com/user-attachments/assets/17eeb7b2-301d-458b-af65-1e1808add765" width="500">

</div>
</div>
???
**Script for Slide: Compare and Swap Efficiency**

"In this slide, we compare two approaches to managing shared resources: one using **locks** and the other using **Compare-And-Swap (CAS)** for atomic operations.

---

### **Top Diagram: Lock-Based Approach**
- Here, **Thread 1** attempts to access the shared data structure but is **blocked** because another thread holds the lock.
- While Thread 1 is blocked, it **wastes time** waiting for the lock to be released.
- The operating system or Java Virtual Machine (JVM) scheduler has to manage this waiting thread, which adds overhead and reduces efficiency.

This approach is expensive because switching between threads (context switching) and waiting for locks consumes CPU resources unnecessarily.

---

### **Bottom Diagram: CAS-Based Approach**
- In the bottom diagram, CAS is used for atomic operations, which are provided directly by the **hardware**.
- Here, **Thread 1** performs the operation without being blocked. Instead of waiting, CAS repeatedly checks and updates the shared variable in a non-blocking manner.
- This eliminates the need for **locks** and avoids expensive thread blocking or waiting states.

---

### **Why CAS Is More Efficient**
- **No Blocking**: Threads don’t get stuck waiting for locks, avoiding wasted time.
- **Fewer Context Switches**: The scheduler doesn’t need to pause and resume threads as often, saving CPU cycles.
- **Hardware Optimization**: CAS leverages atomic hardware instructions, making operations much faster.

In summary, CAS-based atomic operations are far more efficient because they remove the bottleneck of blocking threads and scheduler overhead, resulting in better performance for multi-threaded applications."

---
class: center, middle

# 12 Factor App

???
- IV Backing Services (I should be counting the votes in a database)
- VI Processes should be stateless
- VII Concurrency should be reliable
Share nothing, horizontally partition 12 factor app processes

---

class: center, middle

# Key Takeaways

???
### Key Takeaways (1 minute)
Race conditions can occur in simple operations like voteCount++.
Use synchronized or atomic classes (AtomicInteger) to ensure thread safety.
Concurrency must be reliable and efficient in modern applications.
---
class: center, middle

# Questions?
### https://tomspencerlondon.github.io/concurrency-basics/
???
Thank you for your time! Concurrency is challenging, but understanding the basics helps you prevent these issues early. 
I’ll now take any questions.
