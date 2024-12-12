class: center, middle

# Debugging Concurrency

???
Debugging concurrency in simple applications can help us understand this topic before we meet these issues in production.
---
class: center, middle

# Simple Voting Service

???
We introduce a simple voting scenario: we have proposals, and we increment a vote count each time someone votes.

- simple web service that allows users to vote on proposals.
- When multiple users vote at the same time, we want to ensure that the final vote count is correct
- We want the user to see the real count as it is updated
---
class: center, middle

# The Code Example
???
We will look at the code for the application.
---

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

---
class: center, middle

# The Problem

???
# Race condition problem

- Multiple threads may call `incrementVoteCount()` at the same time.
- If both read the same `voteCount` before incrementing, one increment can overwrite the other.
- Result: lost votes.

---
class: center, middle

# The Test Revealing the Issue

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
---

# Failing Test
```
org.opentest4j.AssertionFailedError: Votes should be 1000 after concurrent voting ==> 
Expected :1000
Actual   :985
```
---

# Thread output
```
11:47:09.730 [pool-1-thread-60] INFO com.example.demo.Proposal -- Before count: 2 After count: 3
11:47:09.730 [pool-1-thread-31] INFO com.example.demo.Proposal -- Before count: 97 After count: 98
11:47:09.730 [pool-1-thread-43] INFO com.example.demo.Proposal -- Before count: 47 After count: 48
11:47:09.730 [pool-1-thread-70] INFO com.example.demo.Proposal -- Before count: 71 After count: 72
11:47:09.730 [pool-1-thread-50] INFO com.example.demo.Proposal -- Before count: 83 After count: 84
11:47:09.730 [pool-1-thread-30] INFO com.example.demo.Proposal -- Before count: 8 After count: 9
```
---
class: center, middle

# Detect duplicate counts
- Parse each log line to find the Before count: X and After count: Y.
- Keep track of which "After" values have been seen + line occurred.
- Same "After" value more than once ==> print suspicious lines
---

# Race Condition Detector Code
```java
public class RaceConditionDetector {
    public static void main(String[] args) throws Exception {
        String logFilePath = "/Users/tspencer/Desktop/websockets/spring-websockets.log";
        Pattern pattern = Pattern.compile("Before count:\\s*(\\d+) After count:\\s*(\\d+)");
        BufferedReader reader = new BufferedReader(new FileReader(logFilePath));
        Map<Integer, List<String>> afterLinesMap = new HashMap<>();
        reader.lines().forEach(line -> {
            Matcher m = pattern.matcher(line);
            if (m.find()) {
                int after = Integer.parseInt(m.group(2));
                afterLinesMap.computeIfAbsent(after, k -> new ArrayList<>()).add(line);
            }
        });
        reader.close();
        
        for (Map.Entry<Integer, List<String>> entry : afterLinesMap.entrySet()) {
            if (entry.getValue().size() > 1) {
                System.out.println("Suspicious duplicate After count: " + entry.getKey());
                for (String suspiciousLine : entry.getValue()) {
                    System.out.println(" - " + suspiciousLine);
                }
            }
        }
    }
}
```
---
class: center, middle

# Output for Race Condition Detector
```bash
Suspicious duplicate After count: 80
 - 11:47:09.730 [pool-1-thread-89] INFO com.example.demo.Proposal -- Before count: 79 After count: 80
 - 11:47:09.730 [pool-1-thread-74] INFO com.example.demo.Proposal -- Before count: 79 After count: 80
Suspicious duplicate After count: 213
 - 11:47:10.002 [pool-1-thread-60] INFO com.example.demo.Proposal -- Before count: 212 After count: 213
 - 11:47:10.002 [pool-1-thread-9] INFO com.example.demo.Proposal -- Before count: 212 After count: 213
```
---
class: center, middle

# Making it Thread-Safe

```java
public synchronized void incrementVoteCount() {
    int before = voteCount;
    voteCount++;
    int after = voteCount;

    logger.info("Before count: " + before + " After count: " + after);
}
```

By marking the method as `synchronized`, we ensure that only one thread can increment the count at a time, preventing lost updates.

---
class: center, middle

# Visualizing Concurrency

<div class="side-by-side" style="display: flex;justify-content: space-evenly;">
<div style="display: flex;flex-direction: column;align-items: center;">
<h2>Multiple Threads</h2>
<img src="https://github.com/user-attachments/assets/57972ea6-3e84-4d3e-8612-5da9a1006418" width="300">
</div>
</div>
???
Multiple threads can now interleave their operations.
---
class: center, middle

# Key Takeaways
- Even a simple increment can cause race conditions when multiple threads are involved.
- Synchronization ensures atomic operations on shared mutable state.
- Understanding this at a small scale prepares us for more complex concurrency challenges.
  ???

---

class: center, middle

# Thank you!
## Questions?
### https://tomspencerlondon.github.io/debugging-concurrency/
