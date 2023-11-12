I wanted to understand how NodeJS is termed single-threaded yet scalable. But that is only true when the work is not [[CPU Bound vs IO Bound#CPU Bound]] - however, this works out fine as most of the [[CPU Bound vs IO Bound#Is a web-server CPU Bound or I/O Bound?|responsibilities of a web-server]] are I/O Bound (which are handled by separate threads via UV lib) and thus being single-threaded does not hinder the performance.

However, I wanted to experiment what happens when we have a CPU intensive task to be performed in the API endpoint in NodeJS & in ASP.NET.
# NodeJS

```js hl:5
const http = require("http");
const server = http.createServer((_, res) => {
  console.time("req");
  console.log("req received by server");
  cpuBound();
  res.end("Hello World");
  console.timeEnd("req");
});
const port = 3000;
server.listen(port);
console.log("listening on " + port);

function cpuBound() {
  let a = 0;
  // 53.444s
  for (let k = 0; k < 100; k++) {
    // 548.749ms
    for (let i = 0; i < 90000000; i++) {
      a = a + Math.tan(a);
    }
  }
}
```

As we can see the CPU bound function `cpuBound()` takes  `~54s` to complete. We have a simple http server which logs when a request is received and also takes note of the time per request.

Now we know that NodeJS is single-threaded - which means the requests will be handled by only 1 thread. So, if we send a request, the output would look like follows

```shell
node server.js
listening on 3000
req received by server
req: 53.444s
```

As expected, the request takes `~54s` to complete. Now, if we send 2 requests simultaneously, notice we only get one `req received by server` console output which is triggered by the first request. Since we are executing a CPU intensive task in the endpoint, and since we have only a single thread, the second request would be waiting until the first request completes and thus the second `req received by server` console output is printed after the first request completes.

```shell hl:3,5
node server.js
listening on 3000
req received by server
req: 54.343s
req received by server
req: 54.397s
```

This is a drawback of being single-threaded - you cannot do any CPU intensive task without compromising on other tasks (other incoming requests)
# ASP.NET

```cs hl:7
// namespace,class & imports excluded for brevity
[HttpGet]
public string Get()
{
	Stopwatch sw = Stopwatch.StartNew();
	Console.WriteLine("request received by server");
	CPUBound();
	sw.Stop();
	Console.WriteLine($"Elapsed : {sw.Elapsed}");
	return "Hello World";
}

void CPUBound()
{
	int a = 0;
	// Elapsed : 00:01:18.3942187
	for (int k = 0; k %3C 100; k++)
	{
		// Elapsed : 00:00:00.7924614
		for (int i = 0; i < 90000000; i++)
		{
			a = (int)(a + Math.Tan(a));
		}
	}
}
```

In ASP.NET, each incoming request is handled by a separate thread, so each thread executes the CPU intensive task independently and thus we can accept multiple requests and process them simultaneously.
### Is execution Parallel or Concurrent?

> tldr; parallel until max thread count then concurrent

My processor has 6 cores & 12 Logical Processors (12 threads) ([AMD Ryzen 5 2600X](https://www.techpowerup.com/cpu-specs/ryzen-5-2600x.c2014)) which means, I can execute `12` threads **<u>truly parallelly</u>**  So, if I receive 12 requests simultaneously, ASP.NET will execute them parallelly across all cores (but it will still have some context switching overhead as there are a lot of things running on my PC along with a ASP.NET web server)

However, when the no. of simultaneous requests increase past 12, say 20, then ASP.NET will spawn 20 threads to handle those 20 requests and each of those 20 threads would fight over CPU time across all cores. As a result, we will notice overall slowness in the application as now the threads will be running concurrently across various cores.
## Code

Repository hosted at <https://github.com/manthan55/NodeJS-vs-ASP.NET>
# References

- [Youtube - When is NodeJS Single-Threaded and when is it Multi-Threaded?](<https://www.youtube.com/watch?v=gMtchRodC2I&ab_channel=HusseinNasser>)

