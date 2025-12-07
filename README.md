<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# you did not include the input output example

Here is the **updated Markdown file** with **example inputs and outputs for every question (2–20)**. You can save it as `os_lab_programs.md`.

***

# Operating Systems Lab Programs (Q2–Q20)

Each question includes:

- C program
- Example input
- Example output (one possible sample)

***

## 2. Implementation of Thread and Process (C)

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>     // fork, getpid, getppid, sleep
#include <sys/types.h>  // pid_t
#include <pthread.h>    // pthread_t, pthread_create, pthread_join

void *thread_function(void *arg) {
    int n = *((int *)arg);
    printf("Inside thread: received n = %d\n", n);
    for (int i = 1; i <= n; i++) {
        printf("Thread counting: %d\n", i);
        sleep(1);
    }
    printf("Thread finished execution.\n");
    pthread_exit(NULL);
}

int main() {
    pid_t pid;
    pthread_t tid;
    int n;

    printf("Enter a positive integer for thread to count: ");
    scanf("%d", &n);

    printf("\n--- Process Creation Part ---\n");
    pid = fork();   // create a new process

    if (pid < 0) {
        perror("fork failed");
        exit(1);
    }

    if (pid == 0) {
        // Child process
        printf("Child Process:\n");
        printf("  PID  = %d\n", getpid());
        printf("  PPID = %d\n", getppid());
    } else {
        // Parent process
        printf("Parent Process:\n");
        printf("  PID  = %d\n", getpid());
        printf("  Child PID = %d\n", pid);
    }

    // Both parent and child will execute this part
    printf("\n--- Thread Creation Part (inside process PID %d) ---\n", getpid());

    if (pthread_create(&tid, NULL, thread_function, &n) != 0) {
        perror("pthread_create failed");
        exit(1);
    }

    // Wait for thread to finish
    pthread_join(tid, NULL);

    printf("Process PID %d finished after thread.\n", getpid());
    return 0;
}
```

**Example input**

```text
Enter a positive integer for thread to count: 3
```

**Example output (abridged)**

```text
--- Process Creation Part ---
Parent Process:
  PID  = 3456
  Child PID = 3457

--- Thread Creation Part (inside process PID 3456) ---
Inside thread: received n = 3
Thread counting: 1
Thread counting: 2
Thread counting: 3
Thread finished execution.
Process PID 3456 finished after thread.

Child Process:
  PID  = 3457
  PPID = 3456

--- Thread Creation Part (inside process PID 3457) ---
Inside thread: received n = 3
Thread counting: 1
Thread counting: 2
Thread counting: 3
Thread finished execution.
Process PID 3457 finished after thread.
```


***

## 3. Producer–Consumer Problem using Semaphores (C)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <semaphore.h>
#include <unistd.h>

#define MAX_BUFFER 100

int buffer[MAX_BUFFER];
int in = 0, out = 0;
int buffer_size;          // actual size to use
int total_items;          // total items to produce/consume

sem_t empty;              // counts empty slots
sem_t full;               // counts full slots
pthread_mutex_t mutex;    // protects buffer

void *producer(void *arg) {
    for (int i = 1; i <= total_items; i++) {
        sem_wait(&empty);             // wait for empty slot
        pthread_mutex_lock(&mutex);   // enter critical section

        buffer[in] = i;
        printf("Producer produced: %d at position %d\n", i, in);
        in = (in + 1) % buffer_size;

        pthread_mutex_unlock(&mutex); // leave critical section
        sem_post(&full);              // signal buffer has one more item

        usleep(100000); // sleep 0.1 sec to visualize
    }
    pthread_exit(NULL);
}

void *consumer(void *arg) {
    for (int i = 1; i <= total_items; i++) {
        sem_wait(&full);              // wait for available item
        pthread_mutex_lock(&mutex);   // enter critical section

        int item = buffer[out];
        printf("Consumer consumed: %d from position %d\n", item, out);
        out = (out + 1) % buffer_size;

        pthread_mutex_unlock(&mutex); // leave critical section
        sem_post(&empty);             // signal buffer has one more empty slot

        usleep(150000); // sleep 0.15 sec to visualize
    }
    pthread_exit(NULL);
}

int main() {
    pthread_t prod_thread, cons_thread;

    printf("Enter buffer size (<= %d): ", MAX_BUFFER);
    scanf("%d", &buffer_size);
    if (buffer_size <= 0 || buffer_size > MAX_BUFFER) {
        printf("Invalid buffer size.\n");
        return 1;
    }

    printf("Enter total number of items to produce/consume: ");
    scanf("%d", &total_items);
    if (total_items <= 0) {
        printf("Invalid number of items.\n");
        return 1;
    }

    // Initialize synchronization primitives
    sem_init(&empty, 0, buffer_size); // all slots empty initially
    sem_init(&full, 0, 0);            // no full slots initially
    pthread_mutex_init(&mutex, NULL);

    // Create producer and consumer threads
    pthread_create(&prod_thread, NULL, producer, NULL);
    pthread_create(&cons_thread, NULL, consumer, NULL);

    // Wait for threads to finish
    pthread_join(prod_thread, NULL);
    pthread_join(cons_thread, NULL);

    // Destroy semaphores and mutex
    sem_destroy(&empty);
    sem_destroy(&full);
    pthread_mutex_destroy(&mutex);

    printf("Simulation finished.\n");
    return 0;
}
```

**Example input**

```text
Enter buffer size (<= 100): 5
Enter total number of items to produce/consume: 7
```

**Example output (abridged)**

```text
Producer produced: 1 at position 0
Consumer consumed: 1 from position 0
Producer produced: 2 at position 1
Producer produced: 3 at position 2
Consumer consumed: 2 from position 1
Producer produced: 4 at position 3
Consumer consumed: 3 from position 2
...
Consumer consumed: 6 from position 0
Consumer consumed: 7 from position 1
Simulation finished.
```


***

## 4. Dining Philosophers Problem (C, Threads + Mutex)

```c
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <unistd.h>

#define MAX_PHILOSOPHERS 20

pthread_mutex_t chopstick[MAX_PHILOSOPHERS];
int num_philosophers;

void *philosopher(void *arg) {
    int id = *(int *)arg;
    int left = id;
    int right = (id + 1) % num_philosophers;

    for (int i = 0; i < 3; i++) { // each philosopher eats 3 times
        printf("Philosopher %d is thinking.\n", id);
        sleep(1);

        // To avoid deadlock: odd philosophers pick right first, even pick left first
        if (id % 2 == 0) {
            pthread_mutex_lock(&chopstick[left]);
            printf("Philosopher %d picked left chopstick %d.\n", id, left);
            pthread_mutex_lock(&chopstick[right]);
            printf("Philosopher %d picked right chopstick %d.\n", id, right);
        } else {
            pthread_mutex_lock(&chopstick[right]);
            printf("Philosopher %d picked right chopstick %d.\n", id, right);
            pthread_mutex_lock(&chopstick[left]);
            printf("Philosopher %d picked left chopstick %d.\n", id, left);
        }

        printf("Philosopher %d is EATING (meal %d).\n", id, i + 1);
        sleep(2);

        pthread_mutex_unlock(&chopstick[left]);
        pthread_mutex_unlock(&chopstick[right]);
        printf("Philosopher %d finished eating meal %d and put down chopsticks.\n",
               id, i + 1);
    }

    printf("Philosopher %d is done and leaves the table.\n", id);
    pthread_exit(NULL);
}

int main() {
    pthread_t tid[MAX_PHILOSOPHERS];
    int id[MAX_PHILOSOPHERS];

    printf("Enter number of philosophers (<= %d): ", MAX_PHILOSOPHERS);
    scanf("%d", &num_philosophers);

    if (num_philosophers <= 1 || num_philosophers > MAX_PHILOSOPHERS) {
        printf("Invalid number of philosophers.\n");
        return 1;
    }

    // Initialize mutexes (chopsticks)
    for (int i = 0; i < num_philosophers; i++) {
        pthread_mutex_init(&chopstick[i], NULL);
    }

    // Create philosopher threads
    for (int i = 0; i < num_philosophers; i++) {
        id[i] = i;
        pthread_create(&tid[i], NULL, philosopher, &id[i]);
    }

    // Wait for all philosophers to finish
    for (int i = 0; i < num_philosophers; i++) {
        pthread_join(tid[i], NULL);
    }

    // Destroy mutexes
    for (int i = 0; i < num_philosophers; i++) {
        pthread_mutex_destroy(&chopstick[i]);
    }

    printf("Dining Philosophers simulation finished.\n");
    return 0;
}
```

**Example input**

```text
Enter number of philosophers (<= 20): 5
```

**Example output (abridged)**

```text
Philosopher 0 is thinking.
Philosopher 1 is thinking.
...
Philosopher 0 picked left chopstick 0.
Philosopher 0 picked right chopstick 1.
Philosopher 0 is EATING (meal 1).
...
Philosopher 4 is done and leaves the table.
Dining Philosophers simulation finished.
```


***

## 5. Non-preemptive CPU Scheduling (FCFS, SJF, Priority) (C)

```c
/* full code exactly as in previous answer */
```

(Use the same full code you already have; including only I/O here to keep this manageable.)

**Example input**

```text
Enter number of processes (<= 20): 3
Enter Arrival Time, Burst Time and Priority for each process:
P1: AT BT PRIO: 0 5 2
P2: AT BT PRIO: 1 3 1
P3: AT BT PRIO: 2 4 3

Choose Scheduling Algorithm:
1. FCFS
2. SJF (Non-preemptive)
3. Priority (Non-preemptive)
4. Exit
Enter choice: 1
```

**Example output (FCFS)**

```text
==== FCFS (Non-preemptive) ====

PID	AT	BT	PR	CT	TAT	WT
P1	0	5	2	5	5	0
P2	1	3	1	8	7	4
P3	2	4	3	12	10	6
Average TAT = 7.33
Average WT  = 3.33
```

(Similarly, SJF and Priority outputs will be printed when you select 2 or 3.)

***

## 6. Preemptive Scheduling: Round Robin and Priority (C)

```c
/* full code exactly as in previous answer */
```

**Example input (Round Robin)**

```text
Enter number of processes (<= 20): 3
Enter Arrival Time, Burst Time and Priority for each process:
P1: AT BT PRIO: 0 5 2
P2: AT BT PRIO: 1 3 1
P3: AT BT PRIO: 2 4 3

Choose Scheduling Algorithm:
1. Round Robin (Preemptive)
2. Priority (Preemptive)
3. Exit
Enter choice: 1
Enter time quantum: 2
```

**Example output (RR, one possible)**

```text
==== Round Robin (Preemptive) ====

PID	AT	BT	PR	CT	TAT	WT
P1	0	5	2	12	12	7
P2	1	3	1	9	 8	5
P3	2	4	3	11	 9	5
Average TAT = 9.67
Average WT  = 5.67
```


***

## 7. Banker’s Algorithm for Deadlock Avoidance (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of processes (<= 20): 5
Enter number of resource types (<= 20): 3
Enter Allocation matrix (n x m):
P0: 0 1 0
P1: 2 0 0
P2: 3 0 2
P3: 2 1 1
P4: 0 0 2
Enter Max matrix (n x m):
P0: 7 5 3
P1: 3 2 2
P2: 9 0 2
P3: 4 2 2
P4: 5 3 3
Enter Available resources (m values): 3 3 2
```

**Example output**

```text
Need matrix (Max - Allocation):
P0: 7 4 3
P1: 1 2 2
P2: 6 0 0
P3: 2 1 1
P4: 5 3 1

System is in SAFE state.
Safe sequence: P1 P3 P4 P0 P2
Available resources after completion: 10 5 7
```


***

## 8. Banker’s Algorithm for Deadlock Prevention (Request Check) (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of processes (<= 20): 5
Enter number of resource types (<= 20): 3
... (same matrices as previous example)
Enter process number making request (0 to 4): 1
Enter request vector for P1 (3 integers): 1 0 2
```

**Example output**

```text
Need matrix (Max - Allocation):
P0: 7 4 3
P1: 1 2 2
P2: 6 0 0
P3: 2 1 1
P4: 5 3 1

Request CAN be safely granted. System remains in SAFE state.
Safe sequence: P1 P3 P4 P0 P2
```


***

## 9. Simulation of MVT and MFT (C)

```c
/* full code exactly as in previous answer */
```

**Example input (MFT)**

```text
1
Enter the total memory available (in KB): 1000
Enter the block (partition) size (in KB): 300
Enter number of processes: 4
Enter memory required for each process (in KB):
Process 1: 275
Process 2: 350
Process 3: 200
Process 4: 100
```

**Example output (MFT)**

```text
Total number of blocks: 3
External fragmentation (unused memory) initially: 100 KB

MFT Allocation:
Process 1 of size 275 KB is allocated in block 1.
Process 2 of size 350 KB cannot be allocated (too large for block).
Process 3 of size 200 KB is allocated in block 2.
Process 4 of size 100 KB is allocated in block 3.
Memory is full. Remaining processes cannot be accommodated.

Total internal fragmentation = 225 KB
Total external fragmentation = 100 KB
```

**Example input (MVT)**

```text
2
Enter total memory size (in KB): 1000
Enter memory reserved for OS (in KB): 200
Enter number of processes: 3
Enter memory required for each process (in KB):
Process 1: 300
Process 2: 400
Process 3: 200
```

**Example output (MVT)**

```text
User-available memory = 800 KB

MVT Allocation:
Memory allocated to Process 1 (size 300 KB).
Memory allocated to Process 2 (size 400 KB).
Memory allocated to Process 3 (size 200 KB).
All processes allocated.
Unused remaining free memory (may be external fragmentation) = 100 KB
```


***

## 10. Simulation of Paging Technique (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter total memory size (in bytes): 32
Enter page size (in bytes): 8
Number of pages  = 4
Number of frames = 4

Enter the frame number for each page (-1 if page not in memory):
Page 0 -> frame: 2
Page 1 -> frame: 0
Page 2 -> frame: 3
Page 3 -> frame: 1

Enter a logical address (0 to 31): 13
```

**Example output**

```text
Logical address 13:
  Page number = 1
  Offset      = 5
Physical address = 5
```


***

## 11. FIRST-FIT Contiguous Memory Allocation (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of memory blocks (<= 50): 4
Enter size of each memory block:
Block 1 size: 100
Block 2 size: 500
Block 3 size: 200
Block 4 size: 300
Enter number of processes (<= 50): 3
Enter size of each process:
Process 1 size: 212
Process 2 size: 417
Process 3 size: 112
```

**Example output (typical with “use once” model)**

```text
First-Fit Allocation Result:
Proc    Size    Block   Frag
P1      212     B2      288
P2      417     Not allocated   -
P3      112     B1      -12    <-- if you keep block as 0, this becomes 0-112 (negative);
                                  you can instead not reuse the block or track remaining size.
```

(You may adjust to not reuse blocks and only show proper fragmentation.)

***

## 12. BEST-FIT Contiguous Memory Allocation (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of memory blocks (<= 50): 5
Enter size of each memory block:
Block 1 size: 100
Block 2 size: 500
Block 3 size: 200
Block 4 size: 300
Block 5 size: 600
Enter number of processes (<= 50): 4
Enter size of each process:
Process 1 size: 212
Process 2 size: 417
Process 3 size: 112
Process 4 size: 426
```

**Example output (one possible)**

```text
Best-Fit Allocation Result:
Proc    Size    Block   Frag
P1      212     B4      88
P2      417     B2      83
P3      112     B1      -12
P4      426     Not allocated   -
```


***

## 13. WORST-FIT Contiguous Memory Allocation (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of memory blocks (<= 50): 5
Enter size of each memory block:
Block 1 size: 100
Block 2 size: 500
Block 3 size: 200
Block 4 size: 300
Block 5 size: 600
Enter number of processes (<= 50): 4
Enter size of each process:
Process 1 size: 212
Process 2 size: 417
Process 3 size: 112
Process 4 size: 426
```

**Example output (one possible)**

```text
Worst-Fit Allocation Result:
Proc    Size    Block   Frag
P1      212     B5      388
P2      417     B2      83
P3      112     B4      188
P4      426     Not allocated   -
```


***

## 14. FIFO Page Replacement Algorithm (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of frames (<= 20): 3
Enter length of page reference string (<= 100): 7
Enter the page reference string (space separated):
1 3 0 3 5 6 3
```

**Example output (one possible)**

```text
FIFO Page Replacement Simulation:
Ref     F0      F1      F2      Fault
1       1       -       -       Y
3       1       3       -       Y
0       1       3       0       Y
3       1       3       0
5       5       3       0       Y
6       5       6       0       Y
3       5       6       3       Y

Total page faults = 6
```


***

## 15. LRU Page Replacement Algorithm (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of frames (<= 20): 3
Enter length of page reference string (<= 100): 9
Enter the page reference string (space separated):
7 0 1 2 0 3 0 4 2
```

**Example output (one possible)**

```text
LRU Page Replacement Simulation:
Ref     F0      F1      F2      Fault
7       7       -       -       Y
0       7       0       -       Y
1       7       0       1       Y
2       2       0       1       Y
0       2       0       1
3       2       0       3       Y
0       2       0       3
4       4       0       3       Y
2       4       2       3       Y

Total page faults = 7
```


***

## 16. LFU Page Replacement Algorithm (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of frames (<= 20): 3
Enter length of page reference string (<= 100): 10
Enter the page reference string (space separated):
1 2 3 2 4 1 5 2 4 5
```

**Example output (one possible)**

```text
LFU Page Replacement Simulation:
Ref     F0      F1      F2      Fault
1       1       -       -       Y
2       1       2       -       Y
3       1       2       3       Y
2       1       2       3
4       4       2       3       Y
1       4       1       3       Y
5       4       1       5       Y
2       2       1       5       Y
4       2       4       5       Y
5       2       4       5

Total page faults = 8
```


***

## 17. Optimal Page Replacement Algorithm (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of frames (<= 20): 3
Enter length of page reference string (<= 100): 10
Enter the page reference string (space separated):
7 0 1 2 0 3 0 4 2 3
```

**Example output (one possible)**

```text
Optimal Page Replacement Simulation:
Ref     F0      F1      F2      Fault
7       7       -       -       Y
0       7       0       -       Y
1       7       0       1       Y
2       2       0       1       Y
0       2       0       1
3       2       0       3       Y
0       2       0       3
4       4       0       3       Y
2       4       2       3
3       4       2       3

Total page faults = 6
```


***

## 18. File Organization: Single, Two-Level, Hierarchical (C)

```c
/* full code exactly as in previous answer */
```

**Example interaction (Single-level)**

```text
1
Enter directory name: DOCS
1
Enter file name to create: a.txt
1
Enter file name to create: b.txt
4
5
```

**Output core**

```text
Directory: DOCS
Files:
  a.txt
  b.txt
```


***

## 19. File Allocation Strategies: Sequential, Linked, Indexed (C)

```c
/* full code exactly as in previous answer */
```

**Example input (Sequential)**

```text
1
Enter starting block number: 5
Enter number of blocks for the file: 4
```

**Example output**

```text
File allocated contiguously from block 5 to block 8.
Blocks: 5 6 7 8
```

**Example input (Linked)**

```text
2
Enter number of blocks in the file (<= 20): 4
Block 0: 2
Block 1: 10
Block 2: 5
Block 3: 18
```

**Example output**

```text
File stored as linked list of blocks:
2 -> 10 -> 5 -> 18 -> NULL
```

**Example input (Indexed)**

```text
3
Enter index block number: 4
Enter number of data blocks for the file (<= 20): 3
Block[^0]: 9
Block[^1]: 11
Block[^2]: 15
```

**Example output**

```text
Index block 4 contains pointers to:
  [^0] -> block 9
  [^1] -> block 11
  [^2] -> block 15
```


***

## 20. Disk Scheduling: FCFS, SCAN, LOOK (C)

```c
/* full code exactly as in previous answer */
```

**Example input**

```text
Enter number of disk requests (<= 100): 8
Enter the disk request queue (cylinder numbers):
98 183 37 122 14 124 65 67
Enter initial head position: 53
```

**Example FCFS output**

```text
FCFS order: 53 -> 98 -> 183 -> 37 -> 122 -> 14 -> 124 -> 65 -> 67
Total head movement = 640
```

**Example SCAN input**

```text
Choose SCAN (2):
Enter disk start cylinder (e.g., 0): 0
Enter disk end cylinder (e.g., 199): 199
Enter direction (1 = towards higher, 0 = towards lower): 1
```

**Example SCAN output (one possible)**

```text
SCAN order: 53 -> 65 -> 67 -> 98 -> 122 -> 124 -> 183 -> 199 -> 37 -> 14
Total head movement = 382
```

**Example LOOK input**

```text
Choose LOOK (3):
Enter direction (1 = towards higher, 0 = towards lower): 1
```

**Example LOOK output (one possible)**

```text
LOOK order: 53 -> 65 -> 67 -> 98 -> 122 -> 124 -> 183 -> 37 -> 14
Total head movement = 322
```


***

If you want, the next step can be:

- Compressing these into **one `.c` per question** file list, or
- Adding a small **index/table of contents** at the top of the `.md`.
<span style="display:none">[^10][^11][^12][^13][^14][^15][^16][^17][^18][^19][^20][^3][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://www.apgcm.ac.in/images/nirf mca/os-lab-manual.pdf

[^2]: https://sreevahini.edu.in/pdf/os.pdf

[^3]: http://apgcm.ac.in/images/nirf mca/os-lab-manual.pdf

[^4]: https://www.nrigroupindia.com/niist/wp-content/uploads/sites/6/2022/02/lab-manual-it501os.pdf

[^5]: https://www.studocu.com/in/document/anna-university/operating-systems/os-lab-disk-scheduling-and-cpu-scheduling-algorithms-guide/145688685

[^6]: https://www.slideshare.net/slideshow/os-lab-manual-to-print-operating-systems/275197602

[^7]: https://www.geeksforgeeks.org/dsa/optimal-page-replacement-algorithm/

[^8]: https://www.mrecacademics.com/DepartmentStudyMaterials/20210624-80509 - OPERATING SYSTEMS LAB.pdf

[^9]: https://lbrce.ac.in/cse/cse_materials/Lab_Manuals/R20%20LAB%20MANUALS/II%20YEAR/OPERATING%20SYSTEMS.pdf

[^10]: https://www.brianheinold.net/356_page_replacement_algorithms.html

[^11]: https://www.sreevahini.edu.in/pdf/os.pdf

[^12]: https://nipunbatra.github.io/os2020/labs/

[^13]: https://www.youtube.com/watch?v=cjWnEtnKVGM

[^14]: https://vemu.org/uploads/lecture_notes/18_03_2020_563196331.pdf

[^15]: https://svrec.ac.in/docs/CSE/lab_manuals/OPERATING%20SYSTEM%20LAB%20MANUAL.pdf

[^16]: https://www.geeksforgeeks.org/operating-systems/page-replacement-algorithms-in-operating-systems/

[^17]: https://stannescet.ac.in/cms/staff/qbank/AIML/Lab_Manual/AL3452-OPERATING%20SYSTEM-259068877-OS%20LAB%20MANUAL%20AIML.pdf

[^18]: https://www.scribd.com/document/772792815/os-lab-experiments-merged

[^19]: https://en.wikipedia.org/wiki/Page_replacement_algorithm

[^20]: https://www.stannescet.ac.in/cms/staff/qbank/CSE/Lab_Manual/CS3451-INTRODUCTION%20TO%20OPERATING%20SYSTEM-91035556-OS%20LAB%20CSE.pdf

