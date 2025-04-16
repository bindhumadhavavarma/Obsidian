
## 🧾 6.9 Summary: Deadlocks

---

### 🧱 What is Deadlock?

- Deadlock occurs when a **set of processes are all blocked**, each waiting for an event that only **another process in the same set can cause**.
- Most commonly involves **resources** (hardware/software) being held while requested by others.
- Another type: **communication deadlock** — when all processes wait for messages, but none are in transit.

---

### 🧠 Key Concepts

- **Safe State:** There exists some execution order in which all processes can complete.
- **Unsafe State:** No guarantee that all processes can finish — may lead to deadlock.
- **Banker’s Algorithm:** Prevents deadlock by denying requests that would lead to unsafe states.

---

### 🛠 Methods of Handling Deadlocks

#### 🔍 Deadlock Avoidance
- Dynamically ensure system remains in **safe state**.
- Requires prior knowledge of **maximum resource needs**.
- Implemented via algorithms like **Banker’s Algorithm**.

#### 🔧 Deadlock Prevention
- **Structurally eliminate** at least one of the four necessary conditions:
  1. **Mutual Exclusion:** Share resources or use spooling.
  2. **Hold and Wait:** Require all resources upfront.
  3. **No Preemption:** Allow forceful resource reclaiming.
  4. **Circular Wait:** Impose a global resource ordering.

---

### 🛰 Beyond Resource Deadlock

#### 💬 Communication Deadlock
- Occurs in message-passing systems.
- Resolved using **timeouts** and **retry logic**.

#### 🔄 Livelock
- Processes are **not blocked** but endlessly retry without progress.
- Can be triggered by overly polite or reactive retry logic.

#### ⏳ Starvation
- A process waits **indefinitely** while others get serviced.
- Preventable with **fair scheduling** (e.g., FCFS).

---

### ✅ Final Takeaway

Deadlock, livelock, and starvation all hinder system progress. Each requires:
- **Clear understanding of conditions**, and
- **Careful design** in scheduling, resource handling, and communication.

---
