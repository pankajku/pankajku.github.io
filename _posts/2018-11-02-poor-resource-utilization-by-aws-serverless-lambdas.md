---
layout: post
title: "Poor Resource Utilization by AWS Serverless Lambdas And How It Impacts Your Applications"
date: 2018-11-02
---

A quick note on the title -- although it refers to only AWS Serverless Lambdas, simply because it is the 800-lb gorilla among serverless infrastructure vendors with broadest adoption among developers, as we will see most of the qualitative observations hold for other cloud function vendors as well. My attempts to make it more agnostic mostly resulted in incomprensible titles, so I begrudgingly settled on this click-baity one.

There is a lot going for [serverless architectures](#serverless-architectures) with application backend written as a collection of simple functions and deployed to the data centers operated by AWS, Google Cloud, Microsoft Azure or any number of such cloud vendors. These functions execute in response to various kinds of events: arrival of an HTTP request, presence of a message in a queue or topic, expiration of a timer, change in a data entity, whatever. The application developer writes these functions, packages them as per Cloud operator's instructions or more likely using the Serverless Framework and then runs a command to deploy. The key benefit of this architecture is freedom from managing servers, even VMs, and typical worries of deployment such as capacity planning, scaling, self-healing and so on.

However the current state of serverless architecture support by major vendors leave a lot to be desired. I'll focus on one aspect in this post -- poor utilization of CPU and memory resources in executing cloud functions and its impact on latency and throughput of serverless backends.

TLDR; Serverless deployment environments execute functions in a runtime instance, typically a containerized process, in a serial manner, meaning only one function call stack is active at any time. If existing runtime instances are already executing the function then a new runtime instance will be activated, incurring the startup and initialization costs. This is not a problem if the function is compute-bound, but for IO and network bound functions, this results in poor use of resources during bursty or high volume traffic, distorts metrics used for billing and places significant burden on processing latency and application throughput.

Read on to better appreciate the qualitative and quantitative aspects of the underlying issues and their implications on serverless applications.

## Simple Experiments

Let us write a simple function, deploy it to a cloud, run client to invoke the function in many different ways and observe carefully. For this experiment I will use Javascript as the programming language, Node as the execution environment, Serverless Framework as packaging and deployment tool, AWS as the deployment cloud and `curl` as the client. These also happen to be the popular choices for develoiping serverless applications. 

Here is a simple `helloWorld` function for AWS deployment:

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
  let sum = 0;
  for (const v of a) sum += v; // sum all the numbers of array a
  await sleepSecs(2);  // relinquish control for 2 seconds

  let result =`${instanceId},${invocationId},${aLength},${sum},Hello World\n`;
  return { statusCode: 200, body: result };
};
```

The above code

* initializes a large array with consecutive integers in code segmaent outside the function to simulate initialization overhead such as parsing library code. This takes around 70 millis on an AWS t2.small VM.
* sleeps for 2 seconds to simulate delay while waiting for network I/I operations.
* adds numbers stored in the large global array to simulate compute overhead. This operation takes around 150 millis first time and goes down to 90 millis after few invocation due to JIT compilation and recency of memory access.
* tracks runtime instances with variable `instanceId` initialized to an UUID.
* tracks function invocations within a runtime instance with monotonically incresing variable `invocationId`.

The memory, compute and wait time simulated in this function may not be representative of your functions, but let us go with these for the purposes of this experiment. You can always fine tune these to your liking by cloning the repository and modifying the code  under `aws-hello` directory.

To measure response time we will invoke the function by running command `curl <url>` and using the bash script `mstime.sh` (in place of builtin command `time` on MacOS), as in `mstime.sh curl <url>` where `<url>` is reported back by the deployment tool:
```
# File mstime.sh
ts=$(gdate +%s%N) ; out=$($@ 2> /dev/null) ; tt=$((($(gdate +%s%N) - $ts)/1000000)) ; echo "$out,$tt"
```

This allows us to have the function output and the response time of multiple concurrent commands in the same line in a single file.

Deploying the above function and invoking it twice reports response times to be 2658 and 2276 milliseconds. 

<pre class="screen">
$ <b>~/bin/mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,1,10000000,49999995000000,Hello World,2658
$ <b>~/bin/mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,2,10000000,49999995000000,Hello World,2276
</pre>

Note that both invocations use the same runtime instanceId `272ca546-2228-4231-aa89-cbb8e8c4efe5` but different invocationIds: `1` and `2`. Higher latency for the first invocation is understandable, for it must include the runtime instance startup and global initialization overheads. This is known as the *slow start* or *cold start* problem, a well known phenomenon with cloud functions, and deserves a separate blog post of its own. For now, let us explore what happens on multiple concurrent clients. Invoke our cloud function 10 times in one go by running the `curl` command in background within a loop and redirecting output to file `out.txt`:

<pre class="screen">
$ <b>for i in `seq 1 10`; do ~/bin/mstime.sh curl https://....execute-api.us-west-2.amazonaws.com/dev/helloWorld >> out.txt & done</b>
...
$ <b>cat out.txt</b>
272ca546-2228-4231-aa89-cbb8e8c4efe5,3,10000000,49999995000000,Hello World,2479
882c8e15-51f5-47a8-a514-faac128708cf,1,10000000,49999995000000,Hello World,2752
9c2f027e-62a1-487b-a368-96263238a376,1,10000000,49999995000000,Hello World,3150
5c48e4f5-176b-45dc-bbad-8c1dd694b7db,1,10000000,49999995000000,Hello World,3160
2e691edb-911f-4cad-98a7-40622dd30ec5,1,10000000,49999995000000,Hello World,3231
8ae594b0-b2da-456d-8213-5de54c79ef48,1,10000000,49999995000000,Hello World,3231
ba4f4a66-e10b-4177-a309-675de7b69166,1,10000000,49999995000000,Hello World,3241
712cd5be-66e4-44ea-a951-060a78fc8e8d,1,10000000,49999995000000,Hello World,3358
d9d03287-414d-47f2-b0da-ad2bec7a846a,1,10000000,49999995000000,Hello World,3366
34d40825-db1c-44ff-8da5-1a5c1d35fdd8,1,10000000,49999995000000,Hello World,3411
</pre>

Observe the output for a moment. What do you notice? The runtime instanceIds are all different and the response times are those corresponding to *cold start*, confirming that a runtime instance is being activated for each invocation.

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
For a fair comparison, I ran this program on a VM in the same AWS region as the lambda deployment and ran the client script, chanigng the URL to `http://<hostname>:3000`. The resulting `out.txt` file shows:
<pre class="screen">
88446b49-cada-4b78-8bb3-c5e3f4d30923,1,10000000,49999995000000,Hello World,2281
88446b49-cada-4b78-8bb3-c5e3f4d30923,2,10000000,49999995000000,Hello World,2377
88446b49-cada-4b78-8bb3-c5e3f4d30923,3,10000000,49999995000000,Hello World,2470
88446b49-cada-4b78-8bb3-c5e3f4d30923,4,10000000,49999995000000,Hello World,2564
88446b49-cada-4b78-8bb3-c5e3f4d30923,5,10000000,49999995000000,Hello World,2652
88446b49-cada-4b78-8bb3-c5e3f4d30923,6,10000000,49999995000000,Hello World,2742
88446b49-cada-4b78-8bb3-c5e3f4d30923,7,10000000,49999995000000,Hello World,2833
88446b49-cada-4b78-8bb3-c5e3f4d30923,8,10000000,49999995000000,Hello World,2920
88446b49-cada-4b78-8bb3-c5e3f4d30923,9,10000000,49999995000000,Hello World,3017
88446b49-cada-4b78-8bb3-c5e3f4d30923,10,10000000,49999995000000,Hello World,3107
</pre>

A few observations are in order: the runtime instanceIds are all same whereas function invocationIds change, implying that the functions are being executed within the same process. In fact you can almost visualize all the ten requests being placed in a queue upon arrival, the first one being taken out for executing the sum loop and then placed on a 2 second wait queue. At this moment the second request is taken out of the queue and then placed on 2 second wait queue after executing the sum loop and so on. The requests enter the wait queue at the interval of around 100 milliseconds, the time for executing the sum loop, and exit after being there for 2 seconds. We can almost predict that it can maintain the same pattern for response time for up to 20 simultaneous requests and can easily support 20 concurrent requests issued at intervals of 100 milliseconds each.

### Even More Concurrent Requests

What would happen if we increase the number of concurrent requests from 10 to, say, 100. Will AWS activate 100 instances? Well, the answer, according to my experiments and observations, is that it will indeed activate 100 instances. The observed response times ranged from 3000 to 5000 milliseconds. Another thing I noticed is that more than half of these instances were deactivated within next 5 minutes. A single node server will most likely lose requests and have very high latencies for many of the requests. A collection of 10 node servers would do much better, while using a fraction of the memory resources.

AWS does have a limit on maximum number of active lambdas within an AWS account. The default value is 1000 but it can be increased.

### A Note On AWS Lambda Billing

```
REPORT RequestId: c958f390-e0c1-11e8-b6f8-8bdbb13543f7	Duration: 2372.13 ms	Billed Duration: 2400 ms Memory Size: 1024 MB	Max Memory Used: 107 MB
```

## Analysis

Single process response times, although somewhat better than those in the first set generated by AWS lambdas, are in the same ball-park range. The biggest difference is that the lambda executions in AWS activates 10 runtime instances and hence, take **10 times more memory**. THIS IS A BIG DEAL. Imagine if you had an application with very high rate of incoming requests and a significant wait time, say an application that acted as a proxy to a slower backend application so that the lambda is mostly waiting for the response from the slower application. This will cause a very large number of lambda runtime instances to be active, taking up a lot more memory compared to an architecture where a much smaller number of processes were active, each handling a good number of requests concurrently. Of course, determining the right number of active instances and concurrent executions in each instance to optimize resource utilization within acceptable latency is tricky, and perhaps the reason why AWS lambdas serve only one request at a time. Still, it reminds of the [CGI](#cgi) days when the web server spawned a new process to serve each HTTP request, incurring significant overhead while processing each request.

There is also the CPU overhead of each runtime activation, but that could get amortized over multiple invocations, especially if new requests keep arriving at a steady rate. However bursty traffic could result in frequent activation and deactivations, putting pressure on CPU utilization and degrading latency for a percentage of requests.

Large number of active runtime instances are also problematic for functions that need to interact with connection oriented services such as RDBMS systems as the service needs to keep track of all active connections.

## What About Other Clouds

<pre class="screen">
61fbd455-4438-44d3-a1d7-ab285b7580fa,1,10000000,49999995000000,Hello World,3432
d6cd58ff-5a71-453a-8070-6908cc843fb4,1,10000000,49999995000000,Hello World,4088
802ceade-47dd-4be7-b2b6-dbe6dbb89668,1,10000000,49999995000000,Hello World,4195
e9ffba27-3b2e-4012-8475-a8c5aad09b4e,1,10000000,49999995000000,Hello World,4621
61fbd455-4438-44d3-a1d7-ab285b7580fa,2,10000000,49999995000000,Hello World,6342
d6cd58ff-5a71-453a-8070-6908cc843fb4,2,10000000,49999995000000,Hello World,7328
3238e492-ccde-4051-8b22-2f4bc3e0f9c7,1,10000000,49999995000000,Hello World,7843
e9ffba27-3b2e-4012-8475-a8c5aad09b4e,2,10000000,49999995000000,Hello World,7850
802ceade-47dd-4be7-b2b6-dbe6dbb89668,2,10000000,49999995000000,Hello World,7851
61fbd455-4438-44d3-a1d7-ab285b7580fa,3,10000000,49999995000000,Hello World,8922
</pre>


## So What Is My Point

The main argument of this post may give the impression that Serverless is a flawed architecture and should be avoided for production deployments. Nothing can be farther from truth. Serverless presents an elegant programming model and highly scalable and manageable deployment model for a large category of backend programs and range of load conditions. My belief is that the current implementations of the deployment infrastructure are in early stages of development and we developers should be aware of these limitations to make the right architectural choices based on expected throughput and latency expectations.

Longer term, my hope is that the infrastructure vendors would fine tune their implementations to optimize resource utilization either automatically or by giving more control knobs to sysadmins and make serverless architecture suitable for even more demanding applications.

References
----------
<a name="serverless-architectures">Serverless Architectures</a>: <https://martinfowler.com/articles/serverless.html>

<a name="cgi">CGI</a>: <https://en.wikipedia.org/wiki/Common_Gateway_Interface>