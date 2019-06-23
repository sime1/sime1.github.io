---
layout: post
title: Creating a job Queue in redis
categories: [ redis ]
---
{% raw %}
Redis is an in-memory data structure store. It's very flexible and can be used
in a number of different ways, such as a cache, a database,or a message broker;
It's really fast, and it can be used to share data across nodes in a distributed
system. In this post, I'm going to desig and implement a job queue for a
distributed batch processing system.

## The scenario

We are developing a data processing system, which receives some kind of data as
input, transforms it, and returns the transformed data. The input data can be
very big, and it can take time to process it; in order to handle multiple
concurrent requests we have a number of worker nodes that perform the actual
data processing. Furthermore, we try to split the input data in multiple pieces,
so that the request can be completed more quickly and we don't waste resources;
at the same time this makes the system able to handle more requests, in a
fashion similar to cooperative multitasking.

However, even after being split, some requests may still take a considerable
amount of time; for this reason, we want to be able to limit the number of
workers that will handle such requests; this way those requests will take longer
to be completed, but on the other hand they won't keep other requests from being
handled.

Finally, there may be more requests than workers, so we want the requests to be
stored until they can be handled (i.e. we cannot use pub/sub); we also want to
be able to assign different priorities to different requests, so that if there
are so many requests that they cannot all be processed at the same time, we will
process higher priority requests first.

So, to recap, the job queue should meet the following requirements:

* It should implement different priorities for the requests;
* It should let us limit the number of sub-requests that can be processed
  concurrently;
* It should let us split single requests into multiple sub-requests

## First attempt: LUA scripts

We will first try to implement the queue using the following data structures:

* a **stream** and a **consumer group** for each request
* a **list** for each level of priority (we will use 100)
* a **string** for each request, used as an atomic counter
* a **pub/sub** channel for each request

### Request streams

Inside the request streams we will insert an entry for each sub-request. Using
consumer groups to read data from the streams helps us accomplish the following:

* support for multiple workers: the workers will read from the consumer group,
  and each of them will receive different messages;
* ability to track the sub-requests: using a stream, we can tell what messages
  are currently being processed, and which workers are processing them;
* ability to recover from worker failures: we can monitor the worker nodes, and
  if for some reason one of them crashes, we can retrieve the message which that
  node was processing and put it back into the stream, so that another node can
  process it instead.

Each of these streams will represent a different request, and will contain the
request ID inside its key

### The request lists

Each one of these lists corresponds to a priority level; every time we make a
request, we will push its ID into one of these lists, based on the priority of
that request.

Having a list for every priority level, workers can issue a single pop command,
passing it all the list from the one with the higher priority to the one with
the lowest; the value returned will be the ID of the request which currently has
the higher priority.

We will use a blocking pop command, with timeout 0, so that if currently there
are no pending requests the worker will just wait for one to become available

### The sub-request counter and the pub/sub channel

The sub-request counter will be used to keep track of how many sub-requests are
currently pending for each request; thanks to it, we will be able to send a
notification to the pub/sub channel once the request has been processed.

### The scripts

Before looking at the scripts, let's take a look at the keys we will use for our
structures:

* `{req}:[prio]`: the lists of request with priority equal to `[prio]`
* `sub:{[id]}`: the stream of subrequests of the request with ID `[id]`
* `counter:{[id]}`: the counter of subrequests for the request with ID `[id]`

To insert the requests into the queue, we use the following two scripts:

```lua
-- KEYS[1] is the id of the request
-- ARGV contains the actual sub-requests

local req_stream = string.format( "sub:{%s}", KEYS[1] )
redis.call("SET", string.format( "counter:{%s}", KEYS[1] ), #ARGV)
-- we need to create the stream before the group
redis.call("XADD", req_stream, "*", "req", "0")
redis.call("XTRIM", req_stream, "MAXLEN", "0")
redis.call("XGROUP", "CREATE", req_stream, req_stream, "$")
for _, r in ipairs(ARGV) do
  redis.call("XADD", req_stream, "*", "req", r)
end
return redis.status_reply("OK")
```

```lua
-- insert the request ID into the request list
--
-- KEYS[1] contains the key of the list with the priority of the request
-- ARGV[1] contains the ID of the request
-- ARGV[2] contains the concurrency limit, i.e. the number of workers that will
-- process sub-requests concurrently
for i = 1, ARGV[2] do
  redis.call("LPUSH", KEYS[1], ARGV[1])
end
return redis.status_reply("OK")
```

What this second script does is basically insert a number of *tokens* inside the
lists. The workers will then acquire these tokens to obtain the ID of the
request that they should process. Since there are a limited number of tokens,
only a limited nummber of workers will get the ID of a single request, allowing
us to limit the maximum number of workers that will concurrently process a
request.

On the workers side, to get the data of a request we should issue a single
`BRPOP` command to all the request lists:

```redis
BRPOP "{req}:0" "{req}:2" ... "{req}:99" 0
```

We use the blocking version so that if there are no requests queued, the worker
waits for one to become available. Once the BRPOP returns a value, we use that
value to read data from the stream:

```redis
XREADGROUP GROUP sub:{[id]} [worker_id] COUNT 1 STREAMS sub:{[id]} >
```

Here `[id]` is the value returned from the previous call and `[worker_id]` is a
string that uniquely identifies the worker amongst other workers.

Once the worker is done processing the request, it calls this script:

```lua
-- this script checks if we processed the whole request, and if so it cleans
-- up all the structures created.
--
-- KEYS[1] contains the ID of the request
-- ARGV[1] contains the ID of the request entry in the consumer group

local req_stream = string.format( "sub:{%s}", KEYS[1] )
local counter = string.format( "counter:{%s}",KEYS[1] )
local remaining = redis.call("DECR", counter)
if remaining == 0 then
  redis.call("XGROUP", "DESTROY", req_stream, req_stream)
  redis.call("DEL", req_stream)
  redis.call("DEL", counter)
  redis.call("PUBLISH", KEYS[1], "1")
else
  redis.call("XACK", req_stream, req_stream, ARGV[1])
end
return redis.status_reply("OK")
```

after this script is executed, the worker will return the *token* it acquired at
the beginning to the list it removed it from:

```redis
LPUSH {req}:[prio] {reqID}
```

reading the redis documentation you should notice that when you issue a `BRPOP`
command to multiple lists, the response contains the ID of the list the message
was popped from; this means that the workers know where the *token* should be
pushed.

A few initial observations:

1. The scripts can definetly be obtimized for production: for instance, the
   comments should be removed. The code is the way it is for the sake of
   clarity.
2. It may seem strange to have two different scripts to insert requests into the
   queue, when we could probably merge the two scripts and do everything in a
   single transaction. However, as you may have guessed based on the keys I
   used, I designed this queue with redis cluster in mind. To make it simple,
   redis cluster requires that all the keys that the script will interact with
   to be known beforehand, and to be stored in the same cluster node. Having the
   lists and the streams on the same node would mean having everything on the
   same node, without really benefitting from redis cluster; for this reason I
   decided to use two scripts to add requests to the queue, so that the priority
   queues are all stored in the same node but the subrequest streams for
   different requests can be stored in different nodes. If redis cluster support
   isn't needed, the scripts can be merged and executed togheter as a single
   transaction.

after the code samples, we will come back to the scripts and make a few more
observations.

## Code samples

In order to demonstrate how it all works, I wrote a couple of scripts.

The first one takes a file in input, and for every line of the file creates a
request. The second one starts some workers that retrieve the sub-requests,
tranform it creating an object from the redis response, and print the resulting
object to the console. They show how to interact with this queue implementation.

You can find the sample scripts, as well as all other lua scripts described in
this post, in this repo:
[https://github.com/sime1/redis-queue](https://github.com/sime1/redis-queue)

## Final observations

There is definetly space for improvement:

* In the code samples I did not cover the recovery from a worker's failure.
  ideally, every worker would implement a something like a tcp or http ping;
  using the hostname (and port in case of tcp) of the worker as consumer in the
  `XREADGROUP` command, we could have another process periodically call `XINFO`
  and subsequently ping all the workers that have pending messages. If any
  worker does not respond, the process can just insert the message back into the
  stream, so that another worker may process it.

  Alternatively, we could have each worker perform the check when it finishes
  processing a request, and if it finds a non-responding worker it `XCLAIM` its
  pending request instead of trying to get another one from the queue.
* We could also stream the subrequests instead of adding them all at the same
  time. This would be especially useful if it takes some time to split the
  original request in multilpe pieces. In this case, we would have a script to
  begin the streaming which would init the counter at 1, a script that adds a
  subrequest to the stream and increases the counter, and a script to finalize
  the stream once all subrequests have been created, which would decrease the
  counter and possibly publish to the channel if the counter reaches 0.

That's it.

Next time i will try to create a redis module that does the same thing. 
{% endraw %}
