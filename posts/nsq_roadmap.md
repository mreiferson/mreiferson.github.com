This document aims to outline what we believe to be the future of NSQ and how we're going to get
there. It's a living document in that we hope it will inspire conversation and debate, ultimately
producing a robust and clear path forward.  The outline is pretty straightforward:

 1. Where are we now?
  * Discovery
  * Durability
  * Delivery Guarantees

 2. Where are we going?
  * Gossip
  * Write Ahead Log
  * Replication

## Where are we now?

NSQ was publicly released on October 9th, 2012. There have been ~740 commits over 16 releases,
[dozens of client libraries][client_libraries] developed for a diverse set of languages, and many
companies have chosen to rely on NSQ as a fundamental piece of their infrastructure.

We couldn't be more happy with the community's growth. It's been rewarding to interact with users,
give talks, and hear about the ways that teams are using NSQ, both with success and failure, and
where they'd like to see improvement.

Additionally, the landscape has changed in the intervening years since we first started working on
NSQ. The proliferation of interest in Go, and its natural application as a language for building
distributed systems, has contributed to an incredible explosion of libraries and projects
[[1]](#footnote_1) that have vastly improved the tool-chest for application developers.

Given all that, it's important to reflect on the state of things, the set of features and use cases
that NSQ currently supports, and be able to outline a roadmap for where we want to be. After all,
without clear direction, we could continue to grind away at bugs and random features but what are
we actually accomplishing?

There are three key areas that are most often discussed regarding NSQ core. Discovery, durability,
and delivery guarantees. What follows in this section is an overview of the pros and cons of the
current design of each.

### Discovery

NSQ succeeded in opening up the possibilities that exist when elastic streams of data can be
discovered by independent groups of consumers in realtime.

We implemented this via `nsqlookupd`, a daemon that maintains an ephemeral mapping of topics to
`nsqd` nodes. We took a pragmatic approach under-the-hood, all of the `nsqd` participating in a
given cluster maintain TCP connections to `N` (typically 2 or 3) `nsqlookupd` and push
topic/channel metadata over the wire.

Because `nsqlookupd` nodes do not coordinate with their peers the availability characteristics are
simple to reason about - just run as many as you need to achieve redundancy.

As usual, there are a few tradeoffs in taking this approach.

The first, and perhaps most obvious, is that operationally you're required to manage an additional
tier of services. This means more moving parts, more operational overhead, and thus more
opportunity for failure (in general).

The second issue only becomes a factor as you scale to many 100s or 1000s of nodes. It's
inefficient (and error prone) to rely on direct connections and simple heartbeats as a failure
detection mechanism at large scale. `nsqadmin` also exhibits related issues, needing to gather
metadata from all nodes as a cluster scales.

### Durability

NSQ stores messages in memory and transparently persists to disk above a high-water mark.

Primarily, this bounds the memory footprint of a given topic during inevitable backlogs (and thus,
roughly, an `nsqd` node's overall memory footprint).

It *wasn't* necessarily designed as a means to provide message durability, although it could
certainly be configured to accomplish that (for example, setting `--mem-queue-size=0`).

Additionally, disk persistence is implemented such that each topic/channel is an independent queue.
This provides a useful property for channels in that distinct consumer groups can degrade without
affecting others. However, the tradeoff is that an `nsqd` will be persisting the *same* message
data numerous times when multiple channels on a topic back up, increasing the burden on the
underlying IO sub-system.

Meanwhile, the write performance of persistent storage has steadily improved, in part due to the
increased availability and adoption of SSD based solutions, and coupled with the fact that the
write characteristics of a message queue are append-only, it seems reasonable to revisit some of
the original design decisions.

### Delivery Guarantees

NSQ guarantees that messages are delivered *at least once*. In spirit, it adheres to the philosophy
outlined in [Life Beyond Distributed Transactions][lbdt] by Pat Helland of Amazon.

However, there's an obvious caveat. Since replication is an exercise left to the user, if you only
`PUB` to a single `nsqd` then a failure of that node would mean messages kept in memory are lost.

Then how do you *actually* implement a system in which messages are guaranteed to be delivered
despite `N` node failures?

One option is to `PUB` messages to multiple `nsqd` (`N + 1`), effectively making it the producer's
responsibility to replicate data. The problem them becomes: how do you handle the *explicit*
duplication of messages?

Well, you de-dupe of course. [Dave Gardner][dave_gardner] (of Hailo fame) has [written
about][dg_blog] implementing an efficient (probabilistic) method that takes advantage of "reverse
bloom filters". This strategy is used in production at Hailo.

Another approach is to structure your consumers to perform [idempotent][idempotent] operations (an
operation that will produce the same results if executed once or multiple times). In this case, the
duplicated messages no longer have a practical impact. However, this isn't always possible and
certainly isn't trivial.

It's also important to weigh the fact that both approaches introduce an explicit inefficiency,
additional operational overhead, and complexity.

## Where are we going?

There's always a tension between keeping NSQ's core simple and focused while still being useful
for a wide range of use cases.

It's also naive to think that after 2 years of experience operating NSQ in production, and
receiving feedback from other users doing the same, we wouldn't be faced with difficult
choices on how to move forward.

Taking a step back, what are the project's goals?

 * simple to understand and operate
 * pragmatic and orthogonal feature set
 * scalable
 * available (no SPOF)
 * resilient to failure

Simply put, if we're able to identify changes that address the aforementioned deficiencies and are
not at odds with those goals, then they are changes worthy of consideration.

Above, we outlined three high level opportunities for improvement:

 1. **Discovery** - the challenge and complexity of deploying `nsqlookupd`/`nsqadmin` at scale

 2. **Durability** - the inefficiency of duplicated messages on disk and the ephemeral nature of
    messages in memory

 3. **Delivery Guarantees** - the difficulty of performing idempotent operations and the
    operational overhead and inefficiency of explicit duplication

Now let's talk about specific approaches that can get us there...

### Gossip

What if the `nsqd` nodes in a cluster participated in a peer-to-peer network and propagated
topic/channel metadata to each other over a lightweight and scalable gossip protocol?

Fortunately, the fine folks at [hashicorp][hashicorp] released [Serf][serf], a library implementing
[SWIM: Scalable Weakly-consistent Infection-style Process Group Membership Protocol][swim].

Gossip accomplishes a few things:

 1. `nsqlookupd` can be deprecated and (eventually) removed entirely (consumers probe *any*
    `nsqd` node for topic discovery).

 2. Improved failure detection with linear scaling characteristics, no longer relying on primitive
    heartbeats.

 3. `nsqd` nodes can gossip topic/channel depths and propagate that information directly to clients
    (for use as a heuristic in distributing `RDY` state).

The implementation is surprisingly straightforward. Despite the eventually consistent nature of
Serf we can improve propagation and convergence by periodically re-gossiping - and use that as an
additional form of liveness.

The challenge is in maintaining the simplicity of bootstrapping a cluster.

### Write Ahead Log

A write-ahead log (WAL) is a technique used to provide atomicity and durability. Before applying a
transaction, you append an entry to a log on non-volatile storage in order to provide a means to
recover from failure.

If `nsqd` implemented a WAL, in lieu of its current overflow-based disk persistence, it would
provide stronger durability and recovery guarantees.

Additionally, it is more efficient to maintain a single log per topic, knowing that *all* messages
are journaled, rather than the current overflow strategy per topic *and* channel.

#### Compaction

The WAL is not without tradeoffs. One of the biggest challenges is how to compact the log. This is
a necessary function of a WAL - computers don't have infinite storage and we want to minimize
recovery time (a direct function of log size).

For some context, one of the aspects of NSQ that makes it easy to write consumers is that `nsqd` is
responsible for maintaining state of which messages have been processed on a given channel. The
client just subscribes and sets `RDY`, responding to messages as they are processed, while `nsqd`
keeps track of state in aggregate.

It should be possible to implement a log compaction strategy based on the "slowest channel". If the
topic is maintaining the log, the channel that is farthest behind (or "slowest") would dictate the
head of the log (and thus the retention point). You can derive depth as a function of those
variables.

This would preserve consumer semantics, making this change effectively transparent.

### Replication

----

1. <a name="footnote_1"></a>[etcd][etcd], [Consul][consul], [Serf][serf], [Terraform][terraform],
[BoltDB][boltdb], [InfluxDB][influxdb], [Fleet][fleet], [Cayley][cayley],
[CockroachDB][cockroachdb], [SkyDNS][skydns], [Docker][docker], [Kubernetes][kubernetes],
[Flynn][flynn], [CoreOS][coreos], etc.

[client_libraries]: http://nsq.io/clients/client_libraries.html
[dave_gardner]: https://twitter.com/davegardnerisme
[dg_blog]: http://www.davegardner.me.uk/blog/2012/11/06/stream-de-duplication/
[idempotent]: http://en.wikipedia.org/wiki/Idempotence
[lbdt]: http://cs.brown.edu/courses/csci2270/archives/2012/papers/weaker/cidr07p15.pdf
[serf]: http://serfdom.io/
[hashicorp]: http://hashicorp.com/
[swim]: http://www.cs.cornell.edu/~asdas/research/dsn02-swim.pdf
[etcd]: https://github.com/coreos/etcd
[consul]: https://consul.io/
[terraform]: https://terraform.io/
[boltdb]: https://github.com/boltdb/bolt
[cayley]: https://github.com/google/cayley
[cockroachdb]: https://github.com/cockroachdb/cockroach
[skydns]: https://github.com/skynetservices/skydns
[docker]: https://docker.com/
[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
[flynn]: https://flynn.io/
[coreos]: https://coreos.com/
[influxdb]: http://influxdb.com/
[fleet]: https://github.com/coreos/fleet