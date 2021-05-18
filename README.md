# starve-free-reader-writers-problem

The reader-writer problems deal with synchronizing multiple processes trying to read or write upon a shared data. The first and second problem provide a solution where either the reader or writer processes can possibly starve. The third readers-writers problem deals with an implementation where neither the reader or the writer process will ever starve.

## Solution idea:

Let's first consider a writer process. First the process will check the number of active and waiting readers. If both, the number of waiting readers and number of active readers are zero, then the it will read away directly. Otherwise, it will wait until a reader process signals and allows that writer to enter.Once the writer process is done, if it is the last writer, then it will signal and allow all the writers "waiting" readers process to enter. Similarily, the last reader process will signal and allow the "waiting" writers processes to enter.

The solution can be visualized by imagining a room and two benches outside the room for waiting reader and writer process. Suppose you are a writer, if there is no reader inside and no reader on the bench, you can enter. If there are readers inside, then the last reader before leaving will allow all the writers on the bench to enter in the room. Similarily, the last writer to leave the room will allow all the readers on the bench to enter.

## The Semaphore

Semaphore has a First-In-First-Out manner of handling the waiting processes. i.e. - the process that call wait first are the ones are the process that are given the access to semaphore first.

Code:

```c++

#include <queue>
using namespace std;

struct Semaphore
{
    int value = 1;
    queue<int> q;

    void wait(int process_id)
    {
        value--;
        if(value < 0)
        {
            q.push(process_id);
            block(); 
            /*  
                It will block the process until it's woken up.
                Non-busy waiting is used here but busy waiting is also possible 
                in case of block() and wakeup() equivalents being not callable or available in the language.
            */
        }
    }
    
    void signal()
    {
        value++;
        if(value <= 0)
        {
            int process_id = q.pop();
            wakeup(process_id); 
            /*  
                It will wakeup the process having the given pid.
            */
        }
    }
};

```

This way, a FIFO semaphore can be implemented. A queue is used to manage the waiting processes. The process gets blocked after pushing itself onto the queue and is woken up in FIFO order when some other process releases the semaphore.

## Global Variables

```c++

Semaphore* in_sem = new Semaphore();
Semaphore* out_sem = new Semaphore();
Semaphore* writer_sem = new Semaphore();

//Why this is initialized to zero will be explained later.
writer_sem->value = 0; 

// Counts how many readers have started reading.
int start_count = 0; 

// Counts how many readers have completed reading.
int complete_count = 0;

// Indicates whether a writing is waiting.
bool is_writer_waiting = false;

```


## Reader Process Code

```c++

// wait on the in_sem semaphore.
in_sem -> wait(process_id); 

//Assuming the process id is available to use in process_id.
//increment start_count since this process will now start reading.
start_count++;

in_sem->signal();

//Read the data. This is the "critical section".

//wait on the out_sem semaphore.
out_sem -> wait(process_id); 

//increment num_completed since we have completed reading.
complete_count++;

if(writer_waiting && start_count == complete_count)
{
    writer_sem -> signal();
}

out_sem -> signal();

```

## Writer Process Code

```c++

in_sem -> wait(process_id);
out_sem -> wait(process_id);

if(start_count == complete_count)
{
    out_sem->signal();
}
else
{
    is_writer_waiting = 1;
    out_sem -> signal();
    writer_sem -> wait();
    is_writer_waiting = 0;
}

in_sem->signal();

```

## Explanation

The starve-free solution works on this method : Any number of readers can simultaneously read the data. A writer will make its presence known once it has started waiting by setting writer_waiting to true.

Once a writer has come, no new process that comes after it can start reading. This is ensured by the writer acquiring the in_sem semaphore and not leaving it until the end of its process. This ensures that any new process that comes after this (be it reader or writer) will be queued up on in_sem. Now, from the looks of it, it might seem like this is a method that is biased towards writer and may lead to starvation of readers. However, this is where the FIFO nature of semaphore comes into picture.

Suppose processes come as RRRWRWRRR. Now, by our method the first three readers will first read. Then, when the next process which is a writer takes the in_sem semaphore, it won't give it up until it's done. Now, suppose by the time this writer is done writing, the next writer has already arrived. However, it will be queued up on the in_sem semaphore. Thus, it won't get to start writing before the process before it, the reader process has completed. Thus, the reader process will get done and then the writer process will again block any more processes from starting and so on.

To explain the writer process further, in case there's no other reader, it doesn't wait at all. Otherwise, it sets writer_waiting to true and waits on the writer_sem semaphore. Also, to make the writer_sem semaphore synchronizable across both the process where one process only executes wait() and other only executes signal(), we initialize it to 0.

To explain the reader process further, it firsts increments the num_started then reads the data and then increments the num_completed. After this, it checks both these variables are equal. If they are and a writer is waiting (found from writer_waiting), it signals the writer_sem semaphore and finishes.

Thus, it will be pretty clear how we manage to make a starve-free solution for the readers-writers problem with a FIFO semaphore. We can say that all processes will be handled in a FIFO manner. Readers will allow other readers in the queue to start reading with them but writers will block all other processes waiting in the queue from executing before it finishes. In this way, we can implement a reader-writer solution where no process will have to indefinitely wait leading to starvation.

## References

- https://arxiv.org/abs/1309.4507
- http://www.cs.umd.edu/~hollings/cs412/s98/synch/synch1.html
- Abraham Silberschatz, Peter B. Galvin, Greg Gagne - Operating System Concepts















