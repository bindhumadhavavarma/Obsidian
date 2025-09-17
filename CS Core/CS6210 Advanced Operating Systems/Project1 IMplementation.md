
Got it! Let me explain the design and what you need to do step by step, without making the changes myself.

[1 tool called]

## 🎯 **Credit Scheduler Design & Implementation Plan**

### **🏗️ Overall Architecture Design:**

The credit scheduler will **reuse the existing runqueue infrastructure** but change the **scheduling logic**:

- **Keep**: Active/expired queue structure, bitmasks, TAILQ lists
- **Change**: Selection algorithm (highest credits instead of highest priority)
- **Add**: Credit tracking, credit charging, credit refill (epoch management)

---

## **📋 Step-by-Step Implementation Plan:**

### **Step 1: Add Global Configuration Variables**
**Where**: `gt_matrix.c` (top level, after `struct timeval tv1;`)

```c
// Global scheduler configuration  
int use_credit_scheduler = 0;  // 0 = priority, 1 = credit
int enable_load_balancing = 0; // 0 = disabled, 1 = enabled
```

### **Step 2: Fix Command Line Parsing**
**Where**: `parse_args()` function in `gt_matrix.c`

**Issues to fix:**
1. Change `&argv[inx][0]` to `argv[inx]` (you're comparing wrong thing)
2. Set the global variables instead of just printing
3. Add safety check for array bounds

```c
// In the -lb case:
enable_load_balancing = 1;

// In the -s case, fix the strcmp:
if(!strcmp(argv[inx], "0")) {
    use_credit_scheduler = 0;
    printf("use priority scheduler\n");
}
else if(!strcmp(argv[inx], "1")) {
    use_credit_scheduler = 1; 
    printf("use credit scheduler\n");
}
```

### **Step 3: Add Credit Fields to uthread_struct_t**
**Where**: `gt_uthread.h` - inside `uthread_struct` definition

**Add these fields** (after `uthread_gid`):
```c
// Credit scheduler fields
int uthread_credits;        // Current remaining credits
int uthread_initial_credits; // Base credits for this thread (25/50/75/100)
struct timeval last_scheduled_time; // For accurate credit charging
```

**Credit group mapping**:
- Group 0 → 25 credits
- Group 1 → 50 credits  
- Group 2 → 75 credits
- Group 3 → 100 credits

### **Step 4: Create Credit Scheduler Functions**
**Where**: `gt_pq.c` - add new functions

**Functions to implement:**

1. **`sched_find_best_uthread_credit(kthread_runqueue_t *kthread_runq)`**
   - Similar to `sched_find_best_uthread()` but select by highest credits
   - Still use active/expired queue swapping for epochs
   - When active queue empty → epoch rollover

2. **`credit_charge_thread(uthread_struct_t *u_obj, int cpu_cycles_used)`**
   - Deduct credits from running thread
   - Move to expired queue if credits <= 0

3. **`credit_refill_epoch(kthread_runqueue_t *kthread_runq)`**
   - Refill all threads in expired queue to initial credits
   - Move them back to active queue

### **Step 5: Modify Scheduler Selection Logic**
**Where**: `gt_kthread.c` - in `ksched_priority()` function

**Current line 269:**
```c
uthread_schedule(&sched_find_best_uthread);
```

**Change to:**
```c
if(use_credit_scheduler) {
    uthread_schedule(&sched_find_best_uthread_credit);
} else {
    uthread_schedule(&sched_find_best_uthread);
}
```

**Problem**: How to access global variables from `gt_kthread.c`?
- **Solution**: Declare them as `extern` in a header file, or pass via shared info structure

### **Step 6: Initialize Credits in Thread Creation**
**Where**: `uthread_create()` in `gt_uthread.c`

**Add credit initialization:**
```c
// Map group ID to credit amount
int credit_amounts[] = {25, 50, 75, 100};
u_new->uthread_initial_credits = credit_amounts[u_gid % 4];
u_new->uthread_credits = u_new->uthread_initial_credits;
gettimeofday(&u_new->last_scheduled_time, NULL);
```

### **Step 7: Add Credit Charging Logic**
**Where**: Timer handler in `gt_kthread.c`

**In `ksched_priority()` before calling `uthread_schedule()`:**
```c
// Charge credits to currently running thread
kthread_context_t *k_ctx = kthread_cpu_map[kthread_apic_id()];
uthread_struct_t *current = k_ctx->krunqueue.cur_uthread;
if(current && use_credit_scheduler) {
    // Calculate time elapsed, charge credits
    credit_charge_thread(current, time_elapsed);
}
```

---

## **🔧 Implementation Order & Tips:**

### **Phase 1 - Basic Structure:**
1. Add global variables
2. Fix argument parsing  
3. Add credit fields to uthread_struct_t
4. Test compilation

### **Phase 2 - Credit Scheduler Core:**
1. Implement `sched_find_best_uthread_credit()`
2. Add scheduler selection logic
3. Initialize credits in thread creation
4. Test basic credit scheduling

### **Phase 3 - Credit Management:**
1. Add credit charging logic
2. Implement epoch rollover
3. Add debug prints for credit queues
4. Test complete credit scheduler

### **Phase 4 - Advanced Features:**
1. Implement `gt_yield()` function
2. Add load balancing
3. Performance measurements

---

## **🚨 Key Design Decisions:**

### **Credit-to-Group Mapping:**
```c
Group 0 (gid % 4 == 0) → 25 credits
Group 1 (gid % 4 == 1) → 50 credits
Group 2 (gid % 4 == 2) → 75 credits  
Group 3 (gid % 4 == 3) → 100 credits
```

### **Epoch Management:**
- **Epoch boundary**: When active queue becomes empty
- **Refill strategy**: Reset all expired threads to initial credits
- **Queue swapping**: Move expired threads back to active

### **Credit Charging:**
- **Timer-based**: Charge on each 100ms tick
- **Proportional**: Charge based on actual time used (for `gt_yield`)
- **Granularity**: 1 credit per timer tick initially

### **Scheduler Selection:**
- **Runtime switching**: Based on command line argument
- **Global variable**: `use_credit_scheduler` flag
- **Function pointer**: Pass different scheduling functions

Would you like me to clarify any part of this design, or shall you start implementing these steps?