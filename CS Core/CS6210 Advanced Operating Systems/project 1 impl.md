
# Credit Scheduler Implementation Report

## Overview and Design Philosophy

Our credit scheduler implementation represents a comprehensive extension to the existing GTThreads user-level threading library, implementing a proportional fair-share CPU scheduler inspired by the Xen Credit Scheduler. The design follows the fundamental principle that CPU time should be allocated proportionally to the credits assigned to each thread, ensuring fairness while maintaining work-conserving properties on symmetric multiprocessing (SMP) systems.
The credit scheduler operates within the existing GTThreads framework, which provides a sophisticated multi-processor user-level threading system. GTThreads creates one kernel thread (kthread) per CPU core, and each kthread maintains its own local runqueue containing user-level threads (uthreads). Our credit scheduler integrates seamlessly with this architecture while introducing credit-based proportional fairness.

### Key Design Principles

1. **Proportional Fairness**: The scheduler guarantees that threads receive CPU time proportional to their allocated credits over time windows called "epochs." This ensures that higher-credit threads get more CPU time while preventing starvation of lower-credit threads.

2. **Work-Conserving Property**: The scheduler never allows a CPU to remain idle while there are runnable threads, maximizing system utilization.

3. **Group-based Credit Assignment**: Credits are assigned based on thread group membership using the predefined values {25, 50, 75, 100}, creating four distinct service classes.

4. **Active/Expired Queue Architecture**: Leverages a sophisticated two-level queue system where threads with remaining credits reside in the active queue, while threads with exhausted credits move to the expired queue until credit refill.

5. **Timer-driven Preemption**: Implements precise time accounting using microsecond-level timestamps and charges threads for actual CPU time consumed.

6. **Load Balancing Integration**: Seamlessly integrates with work-stealing load balancing mechanisms to distribute workload across multiple CPU cores.

7. **Backward Compatibility**: Maintains full compatibility with the existing O(1) priority scheduler, allowing runtime switching between scheduling policies.

## Architecture and Core Components

### 1. Understanding the GTThreads Foundation

Before diving into our credit scheduler implementation, it's essential to understand the underlying GTThreads architecture. GTThreads is a sophisticated user-level threading library that provides multi-processor support through kernel threads (kthreads) mapped to individual CPU cores. Each kthread maintains its own local runqueue and scheduling context, enabling true parallel execution of user-level threads (uthreads).

The original GTThreads library implements an O(1) priority scheduler, which uses bit masks and priority arrays to achieve constant-time scheduling decisions. Our credit scheduler builds upon this foundation while introducing proportional fairness mechanisms.

#### GTThreads Control Flow:
1. **Initialization Phase**: System detects available CPU cores and creates one kthread per core
2. **Thread Creation**: New uthreads are assigned to kthread runqueues using round-robin load distribution
3. **Scheduling Decisions**: Timer-driven preemption (SIGVTALRM) triggers scheduling on the master kthread
4. **Signal Propagation**: Master kthread relays scheduling signals (SIGUSR1) to other kthreads
5. **Context Switching**: Threads save/restore context using sigsetjmp/siglongjmp mechanisms

### 2. Data Structure Enhancements

#### Thread Structure Modifications (`uthread_struct_t`)

Our implementation extends the existing `uthread_struct_t` with additional fields specifically designed for credit-based scheduling and comprehensive performance monitoring:

```c
typedef struct uthread_struct {
    // Original GTThreads fields
    int uthread_state;                      /* Thread state (INIT/RUNNABLE/RUNNING/DONE) */
    int uthread_priority;                   /* Priority level (0-31) */
    int cpu_id;                            /* Current CPU assignment */
    int last_cpu_id;                       /* Previous CPU for migration tracking */
    uthread_t uthread_tid;                 /* Unique thread identifier */
    uthread_group_t uthread_gid;           /* Thread group identifier */
    int (*uthread_func)(void*);            /* Thread function pointer */
    void *uthread_arg;                     /* Thread arguments */
    sigjmp_buf uthread_env;                /* Context save/restore buffer */
    stack_t uthread_stack;                 /* Thread stack information */
    
    // Credit Scheduler Extensions
    int uthread_credits;                   /* Current credits available */
    int uthread_initial_credits;           /* Initial credits assigned at creation */
    struct timeval time_slice_start;       /* Timestamp when thread was last scheduled */
    struct timeval last_scheduled_time;    /* Previous scheduling timestamp */
    struct timeval thread_creation_time;   /* Thread creation timestamp */
    struct timeval total_cpu_time;         /* Accumulated CPU time consumed */
    struct timeval total_wait_time;        /* Accumulated time spent waiting */
    
    TAILQ_ENTRY(uthread_struct) uthread_runq; /* Queue linkage */
} uthread_struct_t;
```

#### Key Design Decisions and Rationale:

**Credit Tracking Fields:**
- `uthread_credits`: Tracks the current credit balance, decremented as the thread consumes CPU time
- `uthread_initial_credits`: Preserves the original credit allocation for epoch-based refill operations
- This dual-tracking approach enables both consumption monitoring and periodic credit restoration

**Temporal Metadata Fields:**
- `time_slice_start`: Records the exact moment when the thread begins execution, enabling precise time charging
- `last_scheduled_time`: Tracks scheduling history for performance analysis and debugging
- `thread_creation_time`: Captures thread lifecycle start for comprehensive performance metrics
- `total_cpu_time`: Accumulates actual CPU time consumed across all scheduling instances
- `total_wait_time`: Tracks time spent in runqueues waiting for CPU allocation

**Performance Monitoring Justification:**
These temporal fields serve multiple purposes:
1. **Accurate Credit Charging**: Precise time measurement ensures fair credit deduction
2. **Performance Analysis**: Detailed timing data supports experimental evaluation
3. **Debugging Support**: Temporal tracking aids in identifying performance bottlenecks
4. **Load Balancing Decisions**: Historical data informs migration policies

#### Credit Assignment Strategy

Credits are assigned during thread creation based on group membership, implementing the project's specified four-tier credit system:

```c
// In uthread_create() function
int credit_amounts[] = {25, 50, 75, 100};
u_new->uthread_credits = credit_amounts[u_gid % 4];
u_new->uthread_initial_credits = u_new->uthread_credits;

// Initialize timing fields
gettimeofday(&u_new->last_scheduled_time, NULL);
gettimeofday(&u_new->thread_creation_time, NULL);
u_new->total_cpu_time.tv_sec = 0;
u_new->total_cpu_time.tv_usec = 0;
u_new->total_wait_time.tv_sec = 0;
u_new->total_wait_time.tv_usec = 0;
```

**Credit Distribution Analysis:**
- **Group 0 (25 credits)**: Represents low-priority background tasks
- **Group 1 (50 credits)**: Standard priority interactive tasks  
- **Group 2 (75 credits)**: High-priority system tasks
- **Group 3 (100 credits)**: Critical real-time tasks

This creates a proportional fairness ratio of approximately 1:2:3:4, ensuring that higher-credit groups receive correspondingly more CPU time over scheduling epochs.

### 3. Queue Management Architecture

#### Detailed Dual Queue System

The credit scheduler cleverly repurposes GTThreads' existing active/expired queue architecture, originally designed for the O(1) priority scheduler, to implement credit-based fairness. This reuse demonstrates the extensibility of the original design while maintaining performance characteristics.

**Queue Architecture Details:**

```c
typedef struct __kthread_runqueue {
    runqueue_t *active_runq;      /* Threads with credits > 0 */
    runqueue_t *expires_runq;     /* Threads with credits = 0 */
    gt_spinlock_t kthread_runqlock; /* Synchronization for queue operations */
    uthread_struct_t *cur_uthread;  /* Currently executing thread */
    uthread_head_t zombie_uthreads; /* Completed threads awaiting cleanup */
    runqueue_t runqueues[2];        /* Physical queue storage */
} kthread_runqueue_t;
```

**Runqueue Internal Structure:**
Each runqueue maintains a sophisticated multi-level organization:

```c
typedef struct __runqueue {
    unsigned int uthread_mask;              /* Bitmask for non-empty priority levels */
    unsigned int uthread_tot;               /* Total threads in queue */
    unsigned int uthread_prio_tot[MAX_UTHREAD_PRIORITY]; /* Threads per priority */
    unsigned int uthread_group_mask[MAX_UTHREAD_GROUPS]; /* Group presence mask */
    unsigned int uthread_group_tot[MAX_UTHREAD_GROUPS];  /* Threads per group */
    prio_struct_t prio_array[MAX_UTHREAD_PRIORITY];      /* Priority level arrays */
} runqueue_t;
```

#### Queue State Transitions and Credit-Based Placement

**Thread Placement Logic:**
Threads are placed in queues based on their credit status, not just their completion status:

1. **Active Queue Placement**: Threads with `uthread_credits > 0` are inserted into the active queue
2. **Expired Queue Placement**: Threads with `uthread_credits == 0` are moved to the expired queue
3. **Queue Maintenance**: Bit masks are updated to reflect queue occupancy for O(1) lookups

**Critical Implementation Detail - Queue Placement During Scheduling:**
```c
// In uthread_schedule() - when a thread is preempted or yields
if (u_obj->uthread_state & (UTHREAD_DONE | UTHREAD_CANCELLED)) {
    // Thread completed - move to zombie queue
    TAILQ_INSERT_TAIL(&(kthread_runq->zombie_uthreads), u_obj, uthread_runq);
} else {
    // Thread still runnable - check credit status
    u_obj->uthread_state = UTHREAD_RUNNABLE;
    if (use_credit_scheduler) {
        if (u_obj->uthread_credits > 0) {
            add_to_runqueue(kthread_runq->active_runq, &(kthread_runq->kthread_runqlock), u_obj);
        } else {
            add_to_runqueue(kthread_runq->expires_runq, &(kthread_runq->kthread_runqlock), u_obj);
        }
    } else {
        // Priority scheduler - always use active queue
        add_to_runqueue(kthread_runq->active_runq, &(kthread_runq->kthread_runqlock), u_obj);
    }
}
```

#### Epoch Transition Mechanism

The scheduler implements an epoch-based credit refresh mechanism that ensures long-term fairness and prevents starvation:

**Epoch Transition Trigger:**
An epoch transition occurs when the active queue becomes completely empty (`!runq->uthread_mask`), indicating that all threads have exhausted their credits.

**Detailed Epoch Transition Process:**
```c
// In sched_find_best_uthread_credit()
if (!runq->uthread_mask) {
    assert(!runq->uthread_tot);  // Verify queue consistency
    
    // Step 1: Refill credits for all threads in expired queue
    credit_refill_epoch(kthread_runq);
    
    // Step 2: Swap queue pointers (O(1) operation)
    kthread_runq->active_runq = kthread_runq->expires_runq;
    kthread_runq->expires_runq = runq;
    
    // Step 3: Update local pointer to new active queue
    runq = kthread_runq->active_runq;
    
    // Step 4: Check if system has any runnable threads
    if (!runq->uthread_mask) {
        assert(!runq->uthread_tot);
        return NULL;  // No more threads to schedule
    }
}
```

**Epoch Properties and Guarantees:**
1. **Bounded Waiting Time**: Every thread is guaranteed to execute within at most two epochs
2. **Proportional Fairness**: Over complete epochs, threads receive CPU time proportional to their credits
3. **Work Conservation**: CPU never idles while runnable threads exist
4. **Starvation Freedom**: Credit refill ensures all threads eventually execute

#### Queue Operation Synchronization

**Thread-Safe Queue Operations:**
All queue operations are protected by per-kthread spinlocks to ensure consistency in the multi-threaded environment:

```c
extern void add_to_runqueue(runqueue_t *runq, gt_spinlock_t *runq_lock, 
                           uthread_struct_t *u_elem) {
    gt_spin_lock(runq_lock);
    runq_lock->holder = 0x02;  // Debug tracking
    __add_to_runqueue(runq, u_elem);  // Internal unlocked operation
    gt_spin_unlock(runq_lock);
}
```

**Lock Granularity Decision:**
We use per-kthread runqueue locks rather than finer-grained locks for several reasons:
1. **Simplicity**: Single lock eliminates complex lock ordering issues
2. **Performance**: Spinlocks provide low overhead for short critical sections
3. **Consistency**: Atomic queue transitions during epoch changes
4. **Debugging**: Single lock holder tracking simplifies debugging

## Core Scheduling Algorithms

### 1. Thread Selection Algorithm - Maximum Credits First Policy

At the heart of our credit scheduler lies a thread selection algorithm that fundamentally differs from traditional priority-based schedulers. Instead of selecting threads based on static priority levels, our implementation adopts a **maximum-credits-first** policy that ensures proportional fairness while maintaining work-conserving properties.

The algorithm begins by examining the active runqueue, which contains all threads that still possess remaining credits. When the scheduler needs to select the next thread to execute, it performs a comprehensive search across all priority levels and thread groups to identify the thread with the highest remaining credit balance. This exhaustive search, while having O(n) time complexity, guarantees optimal fairness within each scheduling epoch by always selecting the thread most deserving of CPU time based on its credit allocation.

A critical aspect of our implementation is the handling of epoch transitions. When the active queue becomes completely empty—indicating that all threads have exhausted their credits—the algorithm triggers an epoch transition. This process involves refilling credits for all threads in the expired queue and swapping the active and expired queues. This mechanism ensures that the scheduler operates in well-defined epochs where proportional fairness is guaranteed over time.

The design deliberately trades O(1) scheduling efficiency for optimal fairness. Unlike the original priority scheduler that uses bit mask operations for constant-time selection, our credit scheduler performs an exhaustive search to find the thread with maximum credits. This trade-off ensures that no thread can gain unfair advantage through lucky queue positioning or timing, as the selection is purely based on remaining credit balance.

### 2. Credit Charging Mechanism - Precise Time Accounting

The foundation of fair resource allocation in our credit scheduler rests on accurate credit charging, which requires precise time measurement and careful arithmetic to ensure that threads are charged exactly for the CPU time they consume. This mechanism operates at microsecond precision to provide fine-grained fairness while maintaining practical efficiency.

When a thread begins execution, the scheduler records a timestamp marking the start of its time slice. Upon preemption, voluntary yield, or thread completion, the charging mechanism calculates the exact elapsed time and converts it to credit units. Our implementation uses a granularity of 10 milliseconds per credit, meaning that threads are charged one credit for every 10 milliseconds of CPU time consumed.

The charging formula employs ceiling division to prevent threads from "stealing" partial time slices. Even if a thread runs for just one millisecond, it is charged a full credit, ensuring that no thread can gain unfair advantage by yielding early or being preempted quickly. This approach maintains fairness by preventing gaming of the scheduling system while keeping the arithmetic simple and efficient.

Beyond credit deduction, the charging mechanism maintains comprehensive timing statistics for each thread, including total CPU time consumed and total waiting time. These statistics serve dual purposes: they enable detailed performance analysis for debugging and optimization, and they provide the data needed for experimental evaluation of the scheduler's effectiveness.

The charging mechanism integrates seamlessly with all preemption points in the system. Whether a thread is preempted by the timer interrupt, voluntarily yields the CPU, or completes its execution, the same charging logic ensures consistent and fair accounting of CPU time consumption.

### 3. Credit Refill Algorithm - Epoch-Based Fairness Restoration

The credit refill mechanism serves as the cornerstone of long-term fairness in our scheduler, ensuring that all threads receive their fair share of CPU time over complete scheduling epochs. This algorithm operates on the principle that fairness should be guaranteed over time windows where all threads have had the opportunity to consume their allocated credits.

An epoch transition occurs when the active queue becomes completely empty, indicating that all currently runnable threads have exhausted their credits. At this point, the refill algorithm systematically traverses the expired queue, restoring every thread to its initial credit allocation. This restoration process ensures that threads begin each new epoch with a clean slate, preventing any long-term accumulation of unfairness.

The refill operation is designed for efficiency and atomicity. Since it operates on the expired queue while the scheduler selects from the active queue, there are no conflicts or race conditions. The algorithm leverages the existing priority and group organization within the queue structure, using bit masks to skip empty sections and process only threads that actually need credit restoration.

After credit refill, the scheduler performs a queue swap operation, promoting the expired queue (now filled with refreshed threads) to become the new active queue, while the empty active queue becomes the new expired queue. This swap operation is performed in constant time through simple pointer manipulation, making epoch transitions very efficient.

The epoch-based approach provides strong theoretical guarantees about fairness and starvation prevention. Every thread is guaranteed to execute within at most two epochs, and over multiple epochs, the actual CPU time allocation converges to the ideal proportional shares defined by the initial credit allocations. This mathematical property ensures that no thread can be permanently disadvantaged, regardless of the timing of its arrival or the behavior of other threads in the system.

## Timer-Based Preemption System

### 1. Signal Infrastructure and Timer Configuration

The credit scheduler integrates seamlessly with GTThreads' existing signal-based preemption infrastructure, which provides the fundamental timing mechanism for both credit charging and scheduling decisions.

#### Timer Initialization and Configuration:

```c
#define KTHREAD_VTALRM_SEC 0
#define KTHREAD_VTALRM_USEC 100000  // 100ms timer intervals

extern void kthread_init_vtalrm_timeslice() {
    struct itimerval timeslice;
    
    // Set recurring timer interval (period between signals)
    timeslice.it_interval.tv_sec = KTHREAD_VTALRM_SEC;
    timeslice.it_interval.tv_usec = KTHREAD_VTALRM_USEC;
    
    // Set initial timer value (time until first signal)
    timeslice.it_value.tv_sec = KTHREAD_VTALRM_SEC;
    timeslice.it_value.tv_usec = KTHREAD_VTALRM_USEC;
    
    // Install virtual timer (ITIMER_VIRTUAL)
    setitimer(ITIMER_VIRTUAL, &timeslice, NULL);
}
```

#### Signal Handler Installation:

```c
// In gtthread_app_init()
kthread_init_vtalrm_timeslice();
kthread_install_sighandler(SIGVTALRM, k_ctx_main->kthread_sched_timer);
kthread_install_sighandler(SIGUSR1, k_ctx_main->kthread_sched_relay);
```

**Design Rationale for Timer Parameters:**
- **100ms Interval**: Balances responsiveness with scheduling overhead
- **Virtual Timer**: Only counts CPU time, not wall-clock time
- **Recurring Timer**: Provides continuous scheduling opportunities
- **Signal-based**: Leverages existing GTThreads preemption infrastructure

### 2. Signal Handler Architecture

#### Master Timer Handler (`ksched_priority`):

The SIGVTALRM signal is delivered to the master kthread (kthread 0), which then orchestrates scheduling decisions across all kthreads:

```c
static void ksched_priority(int signo) {
    kthread_context_t *cur_k_ctx, *tmp_k_ctx;
    int inx;
    
    // Identify current kthread context
    cur_k_ctx = kthread_cpu_map[kthread_apic_id()];
    
    // Relay scheduling signal to all other kthreads
    for (inx = 0; inx < GT_MAX_KTHREADS; inx++) {
        if ((tmp_k_ctx = kthread_cpu_map[inx]) && (tmp_k_ctx != cur_k_ctx)) {
            if (tmp_k_ctx->kthread_flags & KTHREAD_DONE)
                continue;
            // Send SIGUSR1 to trigger scheduling on other kthreads
            syscall(__NR_tkill, tmp_k_ctx->tid, SIGUSR1);
        }
    }
    
    // Perform load balancing (if enabled)
    if (enable_load_balancing && cur_k_ctx->cpuid == 0) {
        kthread_load_balance();
    }
    
    // Handle current thread and schedule next
    uthread_struct_t *current_thread = cur_k_ctx->krunqueue.cur_uthread;
    if (use_credit_scheduler) {
        if (current_thread)
            credit_charge_thread(current_thread);
        uthread_schedule(&sched_find_best_uthread_credit);
    } else {
        uthread_schedule(&sched_find_best_uthread);
    }
}
```

#### Relay Signal Handler (`ksched_cosched`):

Other kthreads receive SIGUSR1 signals and perform their own scheduling decisions:

```c
static void ksched_cosched(int signal) {
    kthread_context_t *cur_k_ctx;
    cur_k_ctx = kthread_cpu_map[kthread_apic_id()];
    
    uthread_struct_t *current_thread = cur_k_ctx->krunqueue.cur_uthread;
    if (use_credit_scheduler) {
        if (current_thread)
            credit_charge_thread(current_thread);
        uthread_schedule(&sched_find_best_uthread_credit);
    } else {
        uthread_schedule(&sched_find_best_uthread);
    }
}
```

### 3. Credit-Specific Timer Integration

#### Precise Time Tracking During Scheduling:

When a thread is selected for execution, its start time is recorded for accurate credit charging:

```c
// In uthread_schedule()
kthread_runq->cur_uthread = u_obj;
if (use_credit_scheduler && u_obj) {
    gettimeofday(&u_obj->time_slice_start, NULL);  // Record start time
}

u_obj->uthread_state = UTHREAD_RUNNING;
siglongjmp(u_obj->uthread_env, 1);  // Transfer control to thread
```

#### Credit Charging at Preemption Points:

The timer-driven charging ensures that threads are charged for actual CPU time consumed:

```c
// Timer fires -> Signal handler invoked -> Current thread charged
if (use_credit_scheduler && current_thread) {
    credit_charge_thread(current_thread);  // Charge for time since time_slice_start
}
```

**Timing Precision Analysis:**
- **Timer Resolution**: 100ms intervals provide regular preemption opportunities
- **Charging Accuracy**: Microsecond precision for actual time measurement
- **Signal Delivery Latency**: Small overhead (~1-10μs) for signal handling
- **Context Switch Cost**: Additional overhead for thread switching

```c
#define KTHREAD_VTALRM_SEC 0
#define KTHREAD_VTALRM_USEC 100000  // 100ms timer intervals

void kthread_init_vtalrm_timeslice() {
    struct itimerval timeslice;
    timeslice.it_interval.tv_sec = KTHREAD_VTALRM_SEC;
    timeslice.it_interval.tv_usec = KTHREAD_VTALRM_USEC;
    setitimer(ITIMER_VIRTUAL, &timeslice, NULL);
}
```

### 2. Timer Handler Integration

The scheduler switches between credit and priority scheduling based on configuration:

```c
static void ksched_priority(int signo) {
    uthread_struct_t *current_thread = cur_k_ctx->krunqueue.cur_uthread;
    if (use_credit_scheduler) {
        if (current_thread)
            credit_charge_thread(current_thread);
        uthread_schedule(&sched_find_best_uthread_credit);
    } else {
        uthread_schedule(&sched_find_best_uthread);
    }
}
```

**Timer Integration Features:**
- **Conditional Switching**: Runtime selection between scheduling policies
- **Time Charging**: Current thread charged before scheduling decision
- **Signal Propagation**: SIGUSR1 signals relayed to other kthreads

### 3. Precise Time Tracking

Each thread tracks its scheduling time for accurate charging:

```c
// In uthread_schedule()
if (use_credit_scheduler && u_obj) {
    gettimeofday(&u_obj->time_slice_start, NULL);
}
```

## Integration with Existing Framework

### 1. Backward Compatibility

The credit scheduler maintains full compatibility with the existing priority scheduler:

- **Shared Data Structures**: Uses existing `runqueue_t` and `uthread_struct_t`
- **Common Interfaces**: Same scheduling function signatures
- **Runtime Selection**: Configurable via `use_credit_scheduler` flag

### 2. Queue Management Reuse

The credit scheduler cleverly repurposes the active/expired queue mechanism:

**Original Priority Scheduler:**
- Active queue: Current priority level threads
- Expired queue: Lower priority or time-expired threads

**Credit Scheduler:**
- Active queue: Threads with remaining credits
- Expired queue: Threads with exhausted credits

### 3. Multi-Core Integration

The scheduler operates seamlessly across multiple kthreads:

```c
extern kthread_runqueue_t *ksched_find_target(uthread_struct_t *u_obj) {
    target_cpu = ksched_info->last_ugroup_kthread[u_gid];
    do {
        target_cpu = ((target_cpu + 1) % GT_MAX_CORES);
    } while (!kthread_cpu_map[target_cpu]);
    
    return (&(kthread_cpu_map[target_cpu]->krunqueue));
}
```

## Load Balancing Integration

### 1. Credit-Aware Migration

The load balancer considers credit status when migrating threads:

```c
static void migrate_thread_to_kthread(uthread_struct_t *thread, 
                                     kthread_context_t *target_k_ctx) {
    if (use_credit_scheduler) {
        if (thread->uthread_credits > 0) {
            add_to_runqueue(target_krunq->active_runq, 
                          &(target_krunq->kthread_runqlock), thread);
        } else {
            add_to_runqueue(target_krunq->expires_runq, 
                          &(target_krunq->kthread_runqlock), thread);
        }
    }
}
```

### 2. Load Balancing Trigger

Load balancing is invoked from the main timer handler:

```c
if (enable_load_balancing && cur_k_ctx->cpuid == 0) {
    kthread_load_balance();
}
```

## Performance and Fairness Analysis

### 1. Scheduling Overhead

**Time Complexity Analysis:**
- Thread Selection: O(n) for exhaustive search
- Credit Charging: O(1) time computation
- Queue Operations: O(1) with proper masking

**Space Complexity:**
- Additional memory per thread: ~40 bytes for time tracking
- No additional queue structures required

### 2. Fairness Guarantees

**Short-term Fairness:**
- Threads with higher credits get priority
- Credit exhaustion forces queue demotion

**Long-term Fairness:**
- Epoch-based credit refill ensures progress
- All threads eventually receive CPU time

### 3. Proportional Share Properties

The scheduler provides proportional fairness based on initial credit allocation:
- Group 0: 25% CPU share (25/150 total credits)
- Group 1: 33% CPU share (50/150 total credits)  
- Group 2: 50% CPU share (75/150 total credits)
- Group 3: 67% CPU share (100/150 total credits)

## Yield Support and Voluntary Preemption

### 1. Enhanced Yield Implementation

The `gt_yield()` function supports both scheduling policies:

```c
extern void gt_yield() {
    if (use_credit_scheduler) {
        credit_charge_thread(cur_uthread);
        if (cur_uthread->uthread_credits > 0) {
            add_to_runqueue(kthread_runq->active_runq, ...);
        } else {
            add_to_runqueue(kthread_runq->expires_runq, ...);
        }
    }
}
```

**Yield Behavior:**
- Voluntary preemption charges for actual CPU time used
- Queue placement based on remaining credits
- Maintains thread state for resumption

## Configuration and Control

### 1. Runtime Configuration

The scheduler supports runtime switching via global flags:

```c
extern int use_credit_scheduler;     // Enable credit-based scheduling
extern int enable_load_balancing;   // Enable load balancing
```

### 2. Parameter Tuning

Key configurable parameters:
- **Timer Interval**: 100ms (KTHREAD_VTALRM_USEC)
- **Credit Values**: {25, 50, 75, 100} per group
- **Time Granularity**: 10ms per credit unit

## Implementation Challenges and Solutions

### 1. Thread Safety

**Challenge**: Multiple kthreads accessing shared scheduling structures
**Solution**: Fine-grained spinlocks with holder identification

```c
gt_spin_lock(&(kthread_runq->kthread_runqlock));
kthread_runq->kthread_runqlock.holder = 0x04;
```

### 2. Time Accuracy

**Challenge**: Precise time measurement for fair charging
**Solution**: Microsecond-resolution timestamps with careful arithmetic

### 3. Credit Starvation

**Challenge**: Threads might exhaust credits and starve
**Solution**: Epoch-based credit refill ensures bounded waiting time

## Testing and Validation

### 1. Matrix Multiplication Workload

The implementation includes comprehensive testing with matrix multiplication:
- Multiple thread groups with different credit allocations
- Performance measurement and timing analysis
- Load balancing effectiveness validation

### 2. Metrics Collection

Comprehensive performance metrics:
```c
struct timeval total_cpu_time;      // Actual CPU time consumed
struct timeval total_wait_time;     // Time spent waiting
struct timeval thread_creation_time; // Thread lifecycle tracking
```

## Conclusion

Our credit scheduler implementation successfully extends the gtthreads framework with a sophisticated proportional-fair scheduling mechanism. The design maintains backward compatibility while providing:

1. **Proportional Fairness**: CPU time allocated based on credit assignments
2. **Efficiency**: Minimal overhead with O(n) scheduling decisions
3. **Integration**: Seamless operation with existing priority scheduler
4. **Scalability**: Multi-core support with load balancing integration
5. **Flexibility**: Runtime configuration and parameter tuning

The implementation demonstrates how credit-based scheduling can be integrated into an existing threading framework while preserving performance and adding significant functionality for resource management and fairness control.

## Voluntary Preemption: The gt_yield() Function

### 1. Design and Implementation

The `gt_yield()` function provides threads with the ability to voluntarily relinquish the CPU, which is essential for cooperative scheduling and fine-grained control over CPU time allocation. Our implementation ensures that yielding threads are charged only for the actual CPU time consumed, maintaining fairness principles.

#### Complete gt_yield() Implementation:

```c
extern void gt_yield() {
    kthread_context_t *k_ctx;
    kthread_runqueue_t *kthread_runq;
    uthread_struct_t *cur_uthread;

    // Get current execution context
    k_ctx = kthread_cpu_map[kthread_apic_id()];
    kthread_runq = &(k_ctx->krunqueue);
    cur_uthread = kthread_runq->cur_uthread;

    if (!cur_uthread) {
        fprintf(stderr, "YIELD: No current thread to yield\n");
        return;  // Nothing to yield
    }

    // For credit scheduler: charge only for actual time used
    if (use_credit_scheduler) {
        credit_charge_thread(cur_uthread);
        fprintf(stderr, "YIELD: Thread %d charged, remaining credits: %d\n",
                cur_uthread->uthread_tid, cur_uthread->uthread_credits);
    }

    // Set thread as runnable (not completed)
    cur_uthread->uthread_state = UTHREAD_RUNNABLE;

    // Queue placement based on scheduler type and credit status
    if (use_credit_scheduler) {
        if (cur_uthread->uthread_credits > 0) {
            add_to_runqueue(kthread_runq->active_runq,
                          &(kthread_runq->kthread_runqlock), cur_uthread);
            fprintf(stderr, "YIELD: Thread %d added to active queue\n",
                    cur_uthread->uthread_tid);
        } else {
            add_to_runqueue(kthread_runq->expires_runq,
                          &(kthread_runq->kthread_runqlock), cur_uthread);
            fprintf(stderr, "YIELD: Thread %d added to expired queue (no credits)\n",
                    cur_uthread->uthread_tid);
        }
    } else {
        // For priority scheduler, always use active queue
        add_to_runqueue(kthread_runq->active_runq,
                      &(kthread_runq->kthread_runqlock), cur_uthread);
    }

    // Clear current thread and save context
    kthread_runq->cur_uthread = NULL;

    // Save current context and schedule next thread
    if (sigsetjmp(cur_uthread->uthread_env, 0)) {
        // Thread resumed after yield
        fprintf(stderr, "YIELD: Thread %d resumed after yield\n",
                cur_uthread->uthread_tid);
        return;
    }

    // Call appropriate scheduler to pick next thread
    if (use_credit_scheduler) {
        uthread_schedule(&sched_find_best_uthread_credit);
    } else {
        uthread_schedule(&sched_find_best_uthread);
    }
}
```

### 2. Credit-Aware Yield Behavior

#### Key Behavioral Differences from Timer Preemption:

**Voluntary vs. Involuntary Preemption:**
1. **Timing Accuracy**: Yield charges for exact time consumed, not timer intervals
2. **Queue Placement**: Credit status determines queue placement (active vs. expired)
3. **Fairness Preservation**: Prevents threads from gaming the system by yielding early
4. **Cooperative Scheduling**: Enables threads to voluntarily improve system responsiveness

#### Credit Charging During Yield:

The yield mechanism ensures that voluntarily preempting threads cannot avoid fair credit charges:

```c
// Before yielding: charge for actual CPU time used
if (use_credit_scheduler) {
    credit_charge_thread(cur_uthread);  // Accurate time-based charging
    
    // Queue placement reflects remaining credit balance
    if (cur_uthread->uthread_credits > 0) {
        // Thread still has credits - eligible for immediate rescheduling
        add_to_runqueue(kthread_runq->active_runq, ...);
    } else {
        // Thread exhausted credits - move to expired queue
        add_to_runqueue(kthread_runq->expires_runq, ...);
    }
}
```

### 3. Integration with Scheduling Policies

#### Context Switching and State Management:

The yield function must carefully manage thread state transitions to ensure correct scheduler behavior:

**State Transition Sequence:**
1. **Current Execution**: Thread is in UTHREAD_RUNNING state
2. **Yield Invocation**: Thread calls gt_yield()
3. **Credit Charging**: Time consumed is calculated and charged
4. **State Change**: Thread state becomes UTHREAD_RUNNABLE
5. **Queue Insertion**: Thread added to appropriate queue based on credits
6. **Context Save**: Thread context saved via sigsetjmp()
7. **Scheduler Invocation**: Next thread selected and executed
8. **Resume Point**: Thread resumes execution when rescheduled

**Context Switching Mechanism:**
```c
// Save current execution context
if (sigsetjmp(cur_uthread->uthread_env, 0)) {
    // Execution resumes here when thread is rescheduled
    return;
}

// Transfer control to scheduler
uthread_schedule(&sched_find_best_uthread_credit);
```

This mechanism leverages the existing GTThreads context switching infrastructure while adding credit-aware behavior.

## Advanced Load Balancing Integration

### 1. Credit-Aware Work Stealing

Our load balancing implementation extends the basic work-stealing algorithm to consider credit status when making migration decisions. This ensures that load balancing operations maintain the fairness properties of the credit scheduler.

#### Complete Load Balancing Implementation:

```c
extern void kthread_load_balance() {
    if (!enable_load_balancing)
        return;

    static int load_balance_count = 0;
    load_balance_count++;

    // Calculate thread distribution across kthreads
    int thread_counts[GT_MAX_KTHREADS];
    int total_threads_system = 0;
    int active_kthreads = 0;

    // Survey current load distribution
    for (int i = 0; i < GT_MAX_KTHREADS; i++) {
        kthread_context_t *k_ctx = kthread_cpu_map[i];
        if (!k_ctx || (k_ctx->kthread_flags & KTHREAD_DONE)) {
            thread_counts[i] = -1;  // Mark as inactive
            continue;
        }

        kthread_runqueue_t *krunq = &(k_ctx->krunqueue);
        thread_counts[i] = krunq->active_runq->uthread_tot + 
                          krunq->expires_runq->uthread_tot;

        if (krunq->cur_uthread != NULL) {
            thread_counts[i]++;  // Count running thread
        }

        total_threads_system += thread_counts[i];
        active_kthreads++;
    }

    int target_per_kthread = total_threads_system / active_kthreads;

    // Perform multiple migration rounds for aggressive balancing
    int migrations_done = 0;
    int max_migrations = 20;

    for (int migration_round = 0; migration_round < max_migrations; migration_round++) {
        kthread_context_t *underloaded_kthread = NULL;
        kthread_context_t *overloaded_kthread = NULL;
        int max_threads = 0;
        int min_threads = total_threads_system;

        // Find underloaded and overloaded kthreads
        for (int i = 0; i < GT_MAX_KTHREADS; i++) {
            kthread_context_t *k_ctx = kthread_cpu_map[i];
            if (!k_ctx || (k_ctx->kthread_flags & KTHREAD_DONE))
                continue;

            kthread_runqueue_t *krunq = &(k_ctx->krunqueue);
            int total_threads = krunq->active_runq->uthread_tot + 
                               krunq->expires_runq->uthread_tot;

            if (krunq->cur_uthread != NULL) {
                total_threads++;
            }

            // Find underloaded kthread (safe to receive migrations)
            if (total_threads < target_per_kthread && 
                total_threads < min_threads &&
                krunq->cur_uthread == NULL) {
                min_threads = total_threads;
                underloaded_kthread = k_ctx;
            }

            // Find overloaded kthread (candidate for migration)
            if (total_threads > target_per_kthread + 1 &&
                total_threads > max_threads) {
                max_threads = total_threads;
                overloaded_kthread = k_ctx;
            }
        }

        // Stop if no beneficial migration possible
        if (!underloaded_kthread || !overloaded_kthread ||
            (max_threads - min_threads) <= 1) {
            break;
        }

        // Attempt thread migration
        uthread_struct_t *migrating_thread = 
            steal_thread_from_kthread(overloaded_kthread);
        if (migrating_thread) {
            migrate_thread_to_kthread(migrating_thread, underloaded_kthread);
            migrations_done++;
        } else {
            break;  // No available threads for migration
        }
    }
}
```

### 2. Credit-Aware Thread Migration

#### Thread Selection for Migration:

```c
static uthread_struct_t *steal_thread_from_kthread(kthread_context_t *k_ctx) {
    kthread_runqueue_t *krunq = &(k_ctx->krunqueue);
    uthread_struct_t *thread = NULL;

    // Try active queue first (threads with credits)
    if (krunq->active_runq->uthread_tot > 0) {
        thread = find_any_thread_in_queue(krunq->active_runq);
        if (thread) {
            rem_from_runqueue(krunq->active_runq, 
                            &(krunq->kthread_runqlock), thread);
        }
    }
    // Then try expired queue (threads without credits)
    else if (krunq->expires_runq->uthread_tot > 0) {
        thread = find_any_thread_in_queue(krunq->expires_runq);
        if (thread) {
            rem_from_runqueue(krunq->expires_runq, 
                            &(krunq->kthread_runqlock), thread);
        }
    }

    return thread;
}
```

#### Credit-Aware Thread Placement After Migration:

```c
static void migrate_thread_to_kthread(uthread_struct_t *thread,
                                     kthread_context_t *target_k_ctx) {
    kthread_runqueue_t *target_krunq = &(target_k_ctx->krunqueue);

    // Update thread's CPU affinity
    thread->last_cpu_id = thread->cpu_id;
    thread->cpu_id = target_k_ctx->cpuid;

    // Place thread in appropriate queue based on credit status
    if (use_credit_scheduler) {
        if (thread->uthread_credits > 0) {
            add_to_runqueue(target_krunq->active_runq,
                          &(target_krunq->kthread_runqlock), thread);
        } else {
            add_to_runqueue(target_krunq->expires_runq,
                          &(target_krunq->kthread_runqlock), thread);
        }
    } else {
        add_to_runqueue(target_krunq->active_runq,
                      &(target_krunq->kthread_runqlock), thread);
    }
}
```

### 3. Load Balancing Performance Impact

#### Balancing Frequency and Overhead:

**Invocation Strategy:**
- Load balancing triggered from master kthread (kthread 0) during timer handler
- Frequency: Every 100ms timer interval when enabled
- Scope: System-wide analysis and migration decisions

**Performance Considerations:**
1. **Migration Cost**: Thread state transfer and queue updates
2. **Cache Locality**: Potential loss of CPU cache affinity
3. **Synchronization Overhead**: Multiple runqueue lock acquisitions
4. **Fairness Preservation**: Credit status maintained across migrations

**Load Balancing Effectiveness:**
- **Idle Prevention**: Eliminates completely idle kthreads
- **Throughput Improvement**: Better resource utilization
- **Fairness Maintenance**: Credit-based placement preserves scheduler properties
- **Scalability**: Algorithm performance scales with number of cores

## Summary: Complete Credit Scheduler Ecosystem

Our credit scheduler implementation represents a comprehensive scheduling solution that successfully integrates multiple advanced features:

### 1. Core Scheduling Components
- **Maximum-Credits-First Selection**: Ensures proportional fairness
- **Epoch-Based Credit Management**: Provides starvation prevention
- **Precise Time Accounting**: Enables accurate resource charging
- **Dual Queue Architecture**: Efficient credit-based thread organization

### 2. Advanced Features
- **Voluntary Preemption**: gt_yield() with credit-aware behavior
- **Work-Stealing Load Balancing**: Credit-preserving thread migration
- **Multi-Core Scalability**: Distributed scheduling across kthreads
- **Runtime Configuration**: Dynamic scheduler selection

### 3. Performance Characteristics
- **Fairness Guarantees**: Long-term proportional CPU allocation
- **Work Conservation**: No CPU idling while threads are runnable
- **Bounded Latency**: Maximum two-epoch waiting time
- **Overhead Management**: Efficient O(n) selection with optimizations

### 4. Integration Quality
- **Backward Compatibility**: Seamless coexistence with priority scheduler
- **Code Reuse**: Leverages existing GTThreads infrastructure
- **Maintainability**: Clean separation of scheduler-specific logic
- **Debugging Support**: Comprehensive logging and state tracking

This implementation demonstrates that sophisticated scheduling policies can be successfully integrated into existing threading frameworks while maintaining performance, correctness, and extensibility.
