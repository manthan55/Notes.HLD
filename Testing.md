We can classify any type of task into majorly 2 categories

- [[#CPU Bound]]
- [[#I/O Bound]]

Other categories includes Memory Bound, GPU Bound, etc.


@slides



> When we say X is **<u>bound</u>** by Y, we mean that it Y is a bottleneck and if we were to improve/upgrade Y, then X can be completed faster
# CPU Bound
Here the task is limited by the speed of the CPU. The faster the CPU, the less time it will take for the task to be completed.
### Examples

^40ee80

- Cryptography
- Massive calculations
- Graphics processing
- Image manipulation
# I/O Bound
Here the task is limited by the speed of the I/O devices. The CPU is extremely fast compared to I/O devices like HDD, RAM or even loading data over the network. So, for any I/O bound task, the CPU is waiting for the I/O device for most of the time. Here, upgrading to a faster CPU will not result in faster task completion. We have to invest in faster I/O devices & higher network bandwidth to be able to improve performance of I/O bound tasks.
### Examples
- Copying/Moving files
- Downloading from internet
- Word processing

# Is a web-server CPU Bound or I/O Bound?
The responsibilities of a web-server are as follow,

| No  | Task                               | CPU task? | I/O task |
| --- | ---------------------------------- | --------- | -------- |
| 1   | Accept request (over n/w)          |           | ✔️       |
| 2   | Deserialize                        |           | ✔️       |
| 3   | Fetch data from Database(over n/w) |           | ✔️       |
| 4   | Process data & request             | ✔️        |          |
| 5   | Serialize                          |           | ✔️       |
| 6   | Return response (over n/w)         |           | ✔️         |

As we can see, most of tasks the web-server performs are I/O related. Thus we can classify a web-server as I/O bound tasks.
# Improving Performance of a CPU Bound task
> tldr; multi-core processing

I earlier said that to improve the performance of a CPU bound task, we have to upgrade the CPU - this is not entirely true. For eg, NodeJS is single threaded - which means it uses a single thread and performs non-blocking I/O with the help of event loop & callbacks. When we launch a NodeJS app/server, we are creating a node process which just has a main thread. I/O operations are however executed by threads thanks to the UV library and callbacks. So, if we are performing CPU intensive tasks, the other requests will not be accepted since there is actually just one thread performing the execution of the CPU intensive task.

However, we can do **multi-core** processing and run a NodeJS process on each core - that way we can make sure we are utilizing all the cores efficiently. Running multiple processes of NodeJS means having multiple instances of our app. This is done with the help of [clusters](https://stackoverflow.com/a/16974684) in Nodejs.

We can also perform multi-threading for CPU bound work as long as the we use exactly the same number of threads as our CPU supports. [[#Multi-Threading with CPU Bound Work]]

# Improving Performance of an I/O Bound task
> tldr; multi-threading

We performing I/O, the CPU is usually waiting for most of the time for the I/O operation to complete. However, instead of upgrading to faster hardware / n/w bandwidth, we can make use of multi-threading. 

For eg, if we want to download a bunch of files, instead of using one thread to download them synchronously (one-after-other), we can assign one download operation to one thread and then the main thread can just wait for all the threads to complete. So now, all the threads are executing asynchronously (not-blocking the main thread) and downloading each file separately.

If downloading 1 file took 10 seconds and we wanted to download 5 files, then it would take,
- 50 seconds to download them in sync (1 thread)
- ~10 seconds to download them async (5 threads)
	- approximately ~10 seconds as need to account for context switching

# Multi-Threading with CPU Bound Work
I did an experiment to understand how multi-threading works in case of CPU Bound work. More details here <https://github.com/manthan55/ThreadingTest>

# References
- [Cannot see context switch overhead for CPU Bound tasks when using ThreadPoolExecutor](https://stackoverflow.com/questions/65370889/cannot-see-context-switch-overhead-for-cpu-bound-tasks-when-using-threadpoolexec)
- [Multithreading: What is the point of more threads than cores?](https://stackoverflow.com/questions/3126154/multithreading-what-is-the-point-of-more-threads-than-cores)
- [Redis is single-threaded, then how does it do concurrent I/O?](https://stackoverflow.com/questions/10489298/redis-is-single-threaded-then-how-does-it-do-concurrent-i-o/10495458#10495458)
- [why Redis is single threaded(event driven)](https://stackoverflow.com/questions/45364256/why-redis-is-single-threadedevent-driven/45374864)