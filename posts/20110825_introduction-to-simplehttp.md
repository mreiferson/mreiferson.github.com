# Introduction to simplehttp

Part of our engineering philosophy is to keep things fast and simple. Aim to serve one purpose and serve it well. Speak HTTP and encode in JSON. Prototype in Python and speed it up in C.

There are a few components that follow these tenets and sit at the core of our infrastructure. We've open-sourced them under the [simplehttp](https://github.com/bitly/simplehttp) moniker. They serve as the the architectural foundation for higher-order functionality.

## simplehttp

At the lowest level is the `simplehttp` library, an abstraction of [libevent](http://monkey.org/~provos/libevent/)'s evhttp functions, aimed at trivializing the task of writing an evented HTTP server in C. It's dead simple yet provides high-level features such as:

 * [Tornado inspired options parsing](https://github.com/bitly/simplehttp/blob/master/simplehttp/options.c)
 * [Tornado inspired logging](https://github.com/bitly/simplehttp/blob/master/simplehttp/log.c)
 * Automatic per-endpoint [stat tracking](https://github.com/bitly/simplehttp/blob/master/simplehttp/stat.c) (request counts, 95% times, averages)
 * Clean API to perform [async HTTP requests](https://github.com/bitly/simplehttp/blob/master/simplehttp/async_simplehttp.c)
 
Built on top of `simplehttp`, perhaps the most important daemons are `simplequeue` and `pubsub`.

### simplequeue

A rock-solid in-memory message queue for arbitrary message bodies (we use JSON), providing basic `/get` and `/put` endpoints. We use this in several key areas to serve as a work queue for asynchronous processing.  During a maintenance window or situations where backend services are degraded it also acts as a buffer, queueing up work that needs to be done when backend services are restored.

We use long-lived Python "queuereaders" to poll the `simplequeue` and perform work.  This work might be writing to a database, logging, aggregating, or anything else that you might not want to perform in a blocking fashion during a request cycle.  These queuereaders have built-in backoff timers which slow down the processing rate when errors are detected to allow a struggling backend to recover gracefully and to reduce the load on the machine running the queuereader.

Generally, we silo a `simplequeue` and its associated queuereaders on each host of the service.  Meaning a `simplequeue` on `hostA` will only contain messages from requests received by that host and its queuereaders will only process messages from its local `simplequeue`.  We do this to address single point of failure issues.

### pubsub

We have many different types of data at bitly, each classified into a stream. There are streams of **encodes** (shortens), **decodes** ("clicks"), **user events**, etc. In order to provide a central, consistent, means for developers to access data in realtime we expose these streams via `pubsub`.

Publishing a message is a simple HTTP request to the `/pub` endpoint, in our case this usually happens in an queuereader who's sole purpose is to read off a specific `simplequeue` and write to a specific `pubsub`.

A client consuming the stream is a a long-lived HTTP request to the `/sub` endpoint.  Messages are transmitted as newline deliminated JSON.

To pair with `pubsub` the repository contains three additional utilities built on `pubsubclient`. `ps_to_file` and `ps_to_http` are fairly self-explanatory. One archives a `pubsub` stream to a file (automatically rolling the output files for you based on a configurable strftime format string), and the other writes a stream of data to destination HTTP endpoints. The latter can be used to send messages to a `simplequeue`, another `pubsub` stream, or any other HTTP endpoint. Additionally, `pubsub_filtered` repeats a pubsub stream and provides the option to remove or obfuscate fields, creating a filtered view on a subset of the data.  At bitly we use these tools to archive our data streams and to pass data published by one application into another application (or another datacenter).

### EOL

If any of these things sound like fun projects to hack on, [bitly is hiring](http://bit.ly/jobs).

<div class="postmeta">
by <a href="http://twitter.com/imsnakes">snakes</a>
</div>