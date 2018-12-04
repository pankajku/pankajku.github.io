---
layout: post
title: "In-Process Concurrency in Serverless Functions And How It Impacts Your Applications"
date: 2018-11-13
---

*The report from Web UI performance group was not good: a handful of pages were taking significantly longer to load, over ten times more than the normal 300 to 500ms. The only thing different for these pages was that they were making a number of requests to populate different parts of the page. However, the requests were concurrent and the backend was architected to be highly scalable*

*A customer reported very high latency and failed requests on a batch job that issued the same API call for all its user accounts.*

The AWS serverless lambda based backend worked perfectly fine under normal conditions or even sustained high load, but came to its knees on sudden burst of requests, despite the periodic no-op warmup requests to keep the lambdas up and running. This prompted me to do a deeper dive. What I found was not what I expected and so this blog post.

There is a lot going for [Serverless Architectures](#serverless-architectures) where an application backend is written as a collection of simple functions and deployed to the data centers operated by AWS, Google Cloud, Microsoft Azure or any number of such cloud vendors. These functions may execute in response to an event: arrival of an HTTP request, presence of a message in a queue or topic, going off of a timer event, change in a persistent data entity, whatever. The application developer writes these functions, packages them as per Cloud operator's instructions or more likely using the tools bundled with [Serverless Framework](#serverless-framework) and then runs a command to deploy. The key benefit of this architecture is freedom from managing physical servers, even cloud hosted VMs, along with typical worries of operations management such as capacity planning, scaling, self-healing and so on.

However the current state of serverless architecture support by major vendors leaves a lot to be desired. I'll focus on one aspect in this post -- lack of in-process concurrency and poor utilization of CPU and memory resources leading to high response times in many common situations.

Here is a summary for the impatient: Serverless deployment environments execute functions in a runtime instance, typically a containerized process, in a serial manner, meaning only one function call stack is active at any time within the process. If existing runtime instances are already executing the function then a new runtime instance must be activated, incurring the startup and initialization costs, irrespective of whether the current executions are actually keeping the CPU busy or merely waiting for I/O. This is not a problem if the function is compute-bound, but for I/O and network bound functions, this results in poor use of resources during bursty or high volume traffic, distorts metrics used for billing, degrades processing latency at client and constricts application throughput.

Note: I wrote most of this post before trying out Azure Cloud Functions and with the assumptions that it will be true in general but was pleasantly surprized by Azure Cloud Functions behavior. Read on for specifics.

## Simple Experiments

Let us write a simple function, deploy it to a AWS, run a client to invoke the function in many different ways and observe the results. For this I will use Javascript as the programming language, Node as the execution environment, the Serverless Framework as packaging and deployment tool, AWS as the deployment cloud and the awesome `curl` as the client. These also happen to be very popular choices for developing serverless applications.

All the code used in this post is available in this [Code Repository](#code-repository) and can be forked, cloned, modified, deployed and executed. As usual, PRs with fixes are most welcome.

Here is a simple `helloWorld` function, written specifically for AWS deployment:

```
// File: aws-hello/index.js
const uuid = require('uuid');

function sleepSecs(secs) {
  return new Promise(resolve => setTimeout(resolve, secs * 1000));
}

let invocationId = 0;
let instanceId = uuid.v4();
let aLength = 1e7; // 10 millions
let a = Array(aLength); // allocate a very large array
for (let i = 0; i < a.length; i++) a[i] = i;

module.exports.helloWorld = async (event, context) => {
  invocationId += 1;
  currInvocationId = invocationId;
  let sum = 0;
  for (const v of a) sum += v; // sum all the numbers of array a
  await sleepSecs(2);  // relinquish control for 2 seconds

  let result =`${instanceId},${currInvocationId},${aLength},${sum},Hello World\n`;
  return { statusCode: 200, body: result };
};
```

As you can see, the above code

* initializes global variable `instanceId` to a UUID value. This value will help us keep track of runtime instance activations.
* initializes a large array of 10 million elements with consecutive integers in code segment outside the function. This simulates cloud function initialization overhead such as parsing and initialization of imported library code. The allocation and initialization of this array takes around 70 ms on an AWS t2.small VM.
* increments global variable `invocationId` and assigns that to local variable `currInvocationId` within the function body to keep track of number of invocations within the same runtime instance.
* sleeps for 2 seconds within the function. This simulates wait time for network I/O.
* adds numbers stored in the large global array to simulate compute overhead. This operation takes around 150 ms first time and goes down to 90 ms after few invocations due to JIT compilation and recency of memory access.
* returns a string with comma separated fields `instanceId`, `invocationId`, array length, sum and the text `Hello World`.


The memory, compute and wait time simulated in this function may not be representative of your functions, but let us go with these for the purposes of this experiment.

To measure response time we will invoke the function by running command `curl <url>` and using the bash script [`mstime.sh`](https://github.com/pankajku/cf-hello-world/blob/master/mstime.sh) (in place of builtin command `time`), as in `mstime.sh curl <url>` where `<url>` is reported back by the deployment tool. This allows us to have the function output and the response time of multiple concurrent commands in the same line in a single file.

Deploying the above function and invoking it twice in quick succession reports response times to be 2658 and 2276 milliseconds. Keep in mind that the exact numbers depend on many factors and are less interesting than their difference.

<pre class="screen">
$ <b>./mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,1,10000000,49999995000000,Hello World,2658
$ <b>./mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,2,10000000,49999995000000,Hello World,2276
</pre>

Both invocations use the same runtime instanceId `272ca546-2228-4231-aa89-cbb8e8c4efe5` but different invocationIds: `1` and `2`. Higher latency for the first invocation is understandable, for it must include the runtime instance startup and global initialization overheads. This is known as the *slow start* or *cold start* problem, a well known phenomenon with cloud functions, and deserves a separate blog post of its own. For now, let us focus on what happens when multiple clients invoke the function at the same time. 

Issue 10 simultaneous requests to our cloud function in one go by running the `curl` command in background within a loop and appending output to file `out.txt`:

<pre class="screen">
$ <b>for i in `seq 1 10`; do ./mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld >> out.txt & done</b>
...
$ <b>cat out.txt</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,3,...,...,...,2479
882c8e15-51f5-47a8-a514-faac128708cf,1,...,...,...,2752
9c2f027e-62a1-487b-a368-96263238a376,1,...,...,...,3150
5c48e4f5-176b-45dc-bbad-8c1dd694b7db,1,...,...,...,3160
2e691edb-911f-4cad-98a7-40622dd30ec5,1,...,...,...,3231
8ae594b0-b2da-456d-8213-5de54c79ef48,1,...,...,...,3231
ba4f4a66-e10b-4177-a309-675de7b69166,1,...,...,...,3241
712cd5be-66e4-44ea-a951-060a78fc8e8d,1,...,...,...,3358
d9d03287-414d-47f2-b0da-ad2bec7a846a,1,...,...,...,3366
34d40825-db1c-44ff-8da5-1a5c1d35fdd8,1,...,...,...,3411
</pre>

What do you notice in the output? The runtime instanceIds are all different and the response times are those corresponding to *cold start*. This suggests that a runtime instance is being activated for each invocation.

### Comparison With Direct Execution

How does it compare with the function directly executed by `node`:
```
// File: node-hello/index.js
const http = require('http');
const uuid = require('uuid');

function sleepSecs(secs) {
  return new Promise(resolve => setTimeout(resolve, secs * 1000));
}

let invocationId = 0;
let instanceId = uuid.v4();
let aLength = 1e7;
let a = Array(aLength);
for (let i = 0; i < a.length; i++) a[i] = i;

const helloWorld = async () => {
  invocationId += 1;
  let currInvocationId = invocationId;
  let sum = 0;
  for (const v of a) sum += v; // sum all the numbers of array a
  await sleepSecs(2);  // relinquish control for 2 seconds

  let result =`${instanceId},${currInvocationId},${aLength},${sum},Hello World\n`;
  return { statusCode: 200, body: result };
};

const handleRequest = async (request, response) => {
  const result = await helloWorld()
  response.writeHead(result.statusCode);
  response.end(result.body);
};
var www = http.createServer(handleRequest);
www.listen(3000);
console.log('Listening on port 3000 ...');
```
For a somewhat fair comparison, I ran this program on a VM in the same AWS region as the lambda deployment and ran the client script on my laptop, chanigng the URL to `http://<hostname>:3000`. The resulting `out.txt` file shows:
<pre class="screen">
88446b49-cada-4b78-8bb3-c5e3f4d30923,1,...,...,...,2281
88446b49-cada-4b78-8bb3-c5e3f4d30923,2,...,...,...,2377
88446b49-cada-4b78-8bb3-c5e3f4d30923,3,...,...,...,2470
88446b49-cada-4b78-8bb3-c5e3f4d30923,4,...,...,...,2564
88446b49-cada-4b78-8bb3-c5e3f4d30923,5,...,...,...,2652
88446b49-cada-4b78-8bb3-c5e3f4d30923,6,...,...,...,2742
88446b49-cada-4b78-8bb3-c5e3f4d30923,7,...,...,...,2833
88446b49-cada-4b78-8bb3-c5e3f4d30923,8,...,...,...,2920
88446b49-cada-4b78-8bb3-c5e3f4d30923,9,...,...,...,3017
88446b49-cada-4b78-8bb3-c5e3f4d30923,10,...,...,...,3107
</pre>

A few observations are in order: the runtime instanceIds are all same whereas function invocationIds increase by one for each invocation, implying that the functions are being executed within the same node process, one after another. Looking at the response times, you can almost visualize all the ten requests being placed in a queue upon arrival, the first one being taken out for executing the sum loop and then placed on a 2 second wait queue. At this moment the second request is taken out of the arrival queue and then placed on 2 second wait queue after executing the sum loop and so on. The requests enter the wait queue at the interval of around 100 milliseconds, the time for executing the sum loop, and exit after being there for 2 seconds. We can almost predict that it can maintain the same pattern for response time for up to 20 simultaneous requests and can easily support 20 concurrent requests issued at intervals of 100 milliseconds each.

**So the single node deployment beats the AWS lambda deployment on latency by using one-tenth of the memory and CPU resources.**

You might argue that each active lambda doesn't keep the CPU busy for whole duration of its execution and perhaps shares the CPU with other processes. True, but as we will see soon AWS charges for the full duration of lambda execution.

### Even More Concurrent Requests

What would happen if we increase the number of concurrent requests from 10 to, say, 100. Will AWS activate 100 instances? Well, the answer, according to my experiments and observations, is that it will indeed activate 100 instances. The observed response times ranged from 3000 to 5000 milliseconds. Another thing I noticed is that more than half of these instances were deactivated within next 5 minutes. A single node server under this kind of load will most likely lose requests and have very high latencies for many of the requests. A collection of 10 node servers would do much better, while using a fraction of the memory resources.

AWS does have a limit on maximum number of active lambdas within an AWS account. The default value is 1000 but it can be increased.

### A Note On AWS Lambda Billing

Here is a log output for one of the invocations indicating billed duration and memory, both allocated and used.
```
REPORT RequestId: c958f390-e0c1-11e8-b6f8-8bdbb13543f7	Duration: 2372.13 ms	Billed Duration: 2400 ms Memory Size: 1024 MB	Max Memory Used: 107 MB
```
The key thing to notice is that the customers are billed for total duration of the lambda execution, _including the wait time_. So for 10 simultaneous requests, the bill would be for 10*2400 = 24,000 ms. This would have been significantly less if all or some of the requests were served by the same runtime instance.

### On Choice of Node For Runtime Environment

Why should we expect concurrent execution of functions within Node when it doesn't support concurrency via threads?

It is true that Node doesn't allow creation of threads but its asynchronous model of execution does allow multiple active function stacks at the same time within a single Node process. In fact Node is designed to support a very large number of open network connections with low overhead.

## Analysis

Single process response times, although somewhat better than those in the first set generated by AWS lambdas, are in the same ball-park range. The biggest difference is that the lambda executions in AWS activates 10 runtime instances and hence, take **10 times more memory**. THIS IS A BIG DEAL. Imagine if you had an application with very high rate of incoming requests and a significant wait time, say an application that acted as a proxy to a slower backend so that the lambda is mostly waiting for the response from the slower application. This will cause a very large number of lambda runtime instances to be active, taking up a lot more memory compared to an architecture where a much smaller number of processes were active, each handling a good number of requests concurrently. Of course, determining the right number of active instances and concurrent executions in each instance to optimize resource utilization within acceptable latency is tricky, and perhaps the reason why AWS lambdas serve only one request at a time. Still, it reminds of the early days of web applications where [CGI standard](#cgi) was used and the web server spawned a new process to serve each HTTP request, incurring significant overhead for each request.

There is also the CPU overhead of each runtime activation, but that could get amortized over multiple invocations, especially if new requests keep arriving at a steady rate. However, bursty traffic could result in frequent activations and deactivations, wasting CPU cycles for avoidable work and degrading latency for a percentage of requests.

Large number of active runtime instances are also problematic for functions that need to interact with connection oriented services such as RDBMS systems as the service maintaining the connections has to allocate and manage resources for each connection.

### What Does AWS Lambda Spec Say

As of this writing [AWS Lambda Programming Model](#aws-lambda-programming-model) is silent about execution environment of lambda functions with respect to concurrency within the same runtime instance. It does recommend reuse of process-local global objects or data structures across multiple invocations but says nothing about in-process locking of global structures. So the implicit assumption is that there won't be concurrent execution of lambdas.

## What About Other Clouds

Here is the output from Google Cloud Function:

<pre class="screen">
61fbd455-4438-44d3-a1d7-ab285b7580fa,1,...,...,...,3432
d6cd58ff-5a71-453a-8070-6908cc843fb4,1,...,...,...,4088
802ceade-47dd-4be7-b2b6-dbe6dbb89668,1,...,...,...,4195
e9ffba27-3b2e-4012-8475-a8c5aad09b4e,1,...,...,...,4621
61fbd455-4438-44d3-a1d7-ab285b7580fa,2,...,...,...,6342
d6cd58ff-5a71-453a-8070-6908cc843fb4,2,...,...,...,7328
3238e492-ccde-4051-8b22-2f4bc3e0f9c7,1,...,...,...,7843
e9ffba27-3b2e-4012-8475-a8c5aad09b4e,2,...,...,...,7850
802ceade-47dd-4be7-b2b6-dbe6dbb89668,2,...,...,...,7851
61fbd455-4438-44d3-a1d7-ab285b7580fa,3,...,...,...,8922
</pre>

As you can see, Google Cloud activated five runtime instances, three of them executed the function twice and one thrice. The second invocations took more than 7000 milliseconds. The third one took almost 9000 ms. Clearly, the function executions within the same runtime instance are sequential.

I also deployed a similar function to Azure. The corresponding output is:

<pre class="screen">
f85ba4c0-1250-4929-bb44-b561afa123b4,1,...,...,...,3800
f85ba4c0-1250-4929-bb44-b561afa123b4,2,...,...,...,4096
f85ba4c0-1250-4929-bb44-b561afa123b4,3,...,...,...,4343
f85ba4c0-1250-4929-bb44-b561afa123b4,4,...,...,...,4560
f85ba4c0-1250-4929-bb44-b561afa123b4,5,...,...,...,4766
f85ba4c0-1250-4929-bb44-b561afa123b4,6,...,...,...,4951
f85ba4c0-1250-4929-bb44-b561afa123b4,7,...,...,...,5154
2fb04f62-feda-48d9-b503-d7f3b375b904,1,...,...,...,8107
1e67eebc-aa04-4d3e-ab1c-8608fbeb4893,1,...,...,...,9393
cfc95039-e915-4c96-aa59-15a3119b65e4,1,...,...,...,10320
</pre>

*This is different.* The first seven invocations are within the same runtime instance and are most likely running _concurrently_. Last three are in different runtime instances. Obviously, Azure is utilizing resources much better.

## A Note on Open Source Implementations

I came across two open source implementations of serverless architecture in course of researching for this post: [Kubeless](#Kubeless) and [Fission](#Fission). These deploy on a [Kubernetes](#Kubernetes) cluster and hence require one to either setup a Kubernetes cluster or have access to one.

I haven't had the time to play with and explore these yet. Will cover them in a separate blog post.

## So What Is My Point

The main argument of this post may give the impression that I view Serverless as a flawed architecture better to be avoided for production deployments. Not at all. Serverless presents an elegant programming model and highly scalable and manageable deployment environment for a large category of backend programs and range of load conditions. My belief is that the current programming model and implementations of the deployment infrastructure are in early stages of development. We developers should be aware of these limitations to make the right architectural choices based on expected throughput and latency expectations.

Longer term my hope is that the infrastructure vendors would fine tune their implementations to optimize latency, throughput and resource utilization either automatically or by giving more control knobs to sysadmins and make serverless architecture suitable for even more demanding applications.

Another thing that is sorely needed is a more concrete specification of cloud function programming and execution model, especially when it comes to concurrent executions within a single process. Ideally, vendors would collaborate so that the same cloud functions can be deployed to any of the compliant clouds.

References
----------
1. <a name="serverless-architectures">Serverless Architectures</a>: <https://martinfowler.com/articles/serverless.html>

2. <a name="serverless-framework">Serverless Framework</a>: <https://serverless.com/framework/>

3. <a name="code-repository">Code Repository</a>: <https://github.com/pankajku/cf-hello-world>

4. <a name="#aws-lambda-programming-model">AWS Lambda Programming Model</a>: <https://docs.aws.amazon.com/lambda/latest/dg/programming-model-v2.html>
   
5. <a name="cgi">CGI</a>: <https://en.wikipedia.org/wiki/Common_Gateway_Interface>

6. <a name="Kubernetes">Kubernetes</a>: <https://kubernetes.io/> 

7. <a name="Fission">Fission</a>: <https://fission.io/>

8. <a name="Kubeless">Kubeless</a>: <https://kubeless.io/>