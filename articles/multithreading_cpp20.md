---
layout: default
---

# C++20: Multithreading

In this article I want to describe all main features which C++ provides for multithreading including changes in C++20.
Let's start by explaining some key concepts in multithreading, such as thread, race conditions, critical section, thread preemption, context switching, and the difference between parallelism and concurrency.

Let's start with the difference between parallelism and concurrency.
Parallelism is a form of execution where two different parts of a program are running at the same time on the CPU, while concurrency refers to execution where the CPU switches between two tasks, giving the illusion that they run simultaneously. So If CPU has only one core and one physical thread then only concurrency is possible. More or less :)

So, what is a thread? I would call it simply a subprogram. What is the difference between a thread and a process? On Linux, threads and processes are very similar, but the main difference is that a process has its own part of virtual memory assigned by the system and cannot share it with another process without using Inter-Process Communication (IPC) mechanisms. Threads, on the other hand, can share the memory of the program with each other by using pointers or simply by accessing global data.
However, this sharing comes with many potential problems. One of them is the race condition.

A race condition occurs when multiple threads read from and/or write to the same data object, which can lead to unexpected results and bugs. We can prevent this using mutexes or atomic operations in C++, but we will discuss that later. If we protect a part of the code against this behavior (e.g., by using a mutex), that part is called a critical section, which means a section of code that can be executed by only one thread at a time.

Modern operating systems use preemptive scheduling, which means the CPU can switch between executing threads.
When a thread is paused (this is called thread preemption), its context is saved (`context` is the state of the registers and other CPU-related data). This leads to what is called a 'context switch', which involves saving the context of one thread and loading the context of another thread to be executed.

Ok so let's spawn a simple thread and check new version of thread introduced in C++20.

```cpp
int main() {
    std::thread t_join([]{ std::cout << "Hello from thread joined\n"; });

    if(t_join.joinable())
    {
        t_join.join();
    }

    std::thread t_detached([]{ std::cout << "Hello from thread detached\n"; });
    t_detached.detach();

    /* SINCE C++20 */
    std::jthread new_thread([](std::stop_token st){ 
        while(!st.stop_requested())
        {
            std::cout << "Hello from jthread\n"; 
        }
    });
    std::stop_source ss = new_thread.get_stop_source();
    ss.request_stop();

    new_thread.request_stop();
}
```
 <!-- spic-lock -->

[back](/)