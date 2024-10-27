# -Round-Robin-Scheduler-with-InterController

This code implements a basic process scheduler using a round-robin approach to manage processes that are enqueued, paused, and resumed. Shared memory and semaphores coordinate between processes, while signals control process state transitions. Let's walk through the functionality, function by function:

### Structure Definitions
- **`Processo` Structure**: This structure holds each process's `pid` and the time it was enqueued, effectively creating nodes for the queue.
- **`FilaProcessos` Structure**: This structure is a circular queue (`fila`), with attributes to track the first and last positions (`primeiro`, `ultimo`) and the current queue size (`tamanho`).
- **`DadosComp` Structure**: This shared data structure holds an instance of `FilaProcessos` and a timestamp marking the program start time (`tempoInicial`). This structure is shared across processes using shared memory.

### Core Scheduler Functions
1. **`inicializarFila(FilaProcessos* fila)`**:
   - Initializes the process queue by setting the first position to 0, the last position to -1 (indicating an empty queue), and size to 0.

2. **`enfileirarProcesso(FilaProcessos* fila, pid_t pid)`**:
   - Adds a process to the queue if there's space. It uses a semaphore to ensure mutual exclusion when modifying the queue.
   - If the queue is full, it prints an error message. Otherwise, it increments `ultimo` (wrapping around to create a circular structure), assigns the process ID and timestamp, and updates the queue size.
   - After enqueuing, it stops the process by sending `SIGSTOP` to its PID.

3. **`desenfileirarProcesso(FilaProcessos* fila, Processo* processo)`**:
   - Removes a process from the front of the queue if it's not empty, copying its details to the provided `processo` pointer.
   - Updates the queue to reflect the removal by moving `primeiro` and decrementing the size. Returns 1 if successful or 0 if the queue is empty.

4. **`buscaProcesso(FilaProcessos* fila, pid_t pid)`**:
   - Searches for a process by its `pid` in the queue. If found, it returns 1; if not, it returns 0. This helps determine if a process is currently enqueued (blocked).

5. **`exibirFila(FilaProcessos* fila, time_t tempoInicial)`**:
   - Displays the queue status, showing each process's position, PID, and wait time.
   - Prints “Fila vazia” if the queue is empty. This function aids in monitoring the queue status for debugging or logging purposes.

6. **`obterIndice(pid_t processo)`**:
   - Locates the index of a given process ID in the `processos` array (an array holding all PIDs of created processes).
   - Returns the index if found, or -1 if the process ID isn’t in the array.

7. **`getNextProcesso(FilaProcessos* fila, pid_t pid)`**:
   - Determines the next process to run by cycling through the `processos` array and finding the first process not currently in the queue.
   - Returns the `pid` of the next process or -1 if no such process is found.

8. **`tratarIRQ0(FilaProcessos* fila)`**:
   - Handles the scheduling signal `SIGALRM`. It checks if the current process is blocked (in the queue).
   - If blocked, it switches to the next process using `getNextProcesso`, stopping the current process and starting the next in line. Updates `processoAtual` (the index of the current running process).
   - If the current process is unblocked, it simply continues without switching.

### Signal Handling Functions
9. **`trataSinal(int signal)`**:
   - Handles the `SIGALRM` signal by attaching to shared memory, invoking `tratarIRQ0` to manage process switching, and detaching afterward.

10. **`IRQ1(int signal)`**:
   - Manages the `SIGUSR1` signal. It dequeues a process that’s been waiting for at least 3 seconds and resumes its execution. Stops the currently running process and makes the dequeued one the active process.
   - Uses shared memory to access the queue.

### Main Execution Logic in `main()`
1. **Shared Memory and Semaphore Initialization**:
   - Sets up shared memory (`shmid`) and a semaphore (`semid`) to synchronize access to shared resources, initializing the semaphore to 1.

2. **Kernel Process**:
   - Forks a child as the "kernel" process, which manages the lifecycle of the processes and scheduling.
   - Inside the kernel:
     - Forks `NUM_PROCESSOS` processes. Each child runs for a specified time, printing its status periodically and enqueuing itself after every 3 seconds of runtime.
     - Stops each child initially using `SIGSTOP`.
     - Sets up signal handlers for `SIGALRM` and `SIGUSR1` to control process switching.
     - Starts the round-robin execution by resuming the first created process.

3. **InterController Process**:
   - Forks another child as the "InterController" process, which periodically triggers process scheduling by sending `SIGALRM` to the kernel.
   - It also verifies if any processes in the queue have waited for 3 seconds, sending `SIGUSR1` to the kernel to dequeue and resume them if they have.

4. **Process Termination and Cleanup**:
   - The main process waits for both kernel and InterController processes to terminate.
   - Cleans up shared memory and semaphore resources to avoid memory leaks.

### Key Elements of Operation
- **Scheduling**: The round-robin approach is implemented through `SIGALRM`-triggered scheduling and `SIGUSR1`-triggered queue management.
- **Synchronization**: The semaphore ensures only one process manipulates the queue at any given time.
- **Concurrency Management**: Shared memory allows data sharing across processes, while signals control process states (start/stop).
