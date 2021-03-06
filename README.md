Gryadka is a minimalistic Paxos-based master-master replicated consistent key/value layer on top of multiple 
instances of Redis. Using 2N+1 Redis instances Gryadka keeps working when up to N instances become 
unavailable.

Its core has less than 500 lines of code but provides full featured Paxos implementation supporting such advance 
features as cluster membership change (ability to add/remove nodes to a cluster) and distinguished proposer optimization 
(using one round trip to change a value instead of two).

#### Is it production ready?

No, it's an educational project and was never intended to be deployed in production. Its goal is to check if it's possible to avoid common pitfalls of implementing a consensus (master-master replication) protocol.

Paxos's author, Leslie Lamport wrote that 
["it is among the simplest and most obvious of distributed algorithms"](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)
but many who tried to implement it run into troubles:

  * ["building a production system turned out to be a non-trivial task for a variety of reasons"](https://www.cs.utexas.edu/users/lorenzo/corsi/cs380d/papers/paper2-1.pdf)
  * ["Paxos is by no means a simple protocol, even though it is based on relatively simple invariants"](http://www.cs.cornell.edu/courses/cs7412/2011sp/paxos.pdf)
  * ["we found few people who were comfortable with Paxos, even among seasoned researchers"](https://raft.github.io/raft.pdf)

This dissonance is the motivation to set the limit on Gryadka's size to force it to be simple.

Even though Gryadka is an educational project, its performance is decent and only 10% behind Etcd (see [the comparison](http://rystsov.info/2017/02/15/simple-consensus.html)).

#### Is it correct?

Yes, it seems so. 

The protocol behind Gryadka (a modification of Paxos) has [an informal proof](http://rystsov.info/2015/09/16/how-paxos-works.html#math1)
. Tobias Schottdorf [explored](https://tschottdorf.github.io/single-decree-paxos-tla-compare-and-swap) it with TLA+ and Greg Rogers wrote [a TLA+ specification](https://medium.com/@grogepodge/tla-specification-for-gryadka-c80cd625944e) (kudos!).

The implementation was heavily tested with fault injections on the network layer.

# Examples

Please see the https://github.com/gryadka/js-example repository for an example of web api built on top of Gryadka and 
its end-to-end consistency testing.

# API

Gryadka's core interface is a `changeQuery` function which takes three arguments:
  
  * a `key`
  * a `change` function
  * a `query` function

Internally `changeQuery` gets a value associated with the `key`, applies `change` to calculate a new value, 
saves it back and returns `query` applied to that new value.

The pseudo-code:

```javascript
class Paxos {
  constuctor() {
    this.storage = ...;
  }
  changeQuery(key, change, query) {
    const value = change(this.storage.get(key));
    this.storage.set(key, value);
    return query(value);
  }
}
```

By choosing the appropriate change/query functions it's possible to customize Gryadka to fulfill different tasks. 
A "last write win" key/value could be implemented as:

```javascript
class LWWKeyValue {
  constuctor(paxos) {
    this.paxos = paxos;
  }
  read(key) {
    return this.paxos.changeQuery(key, x => x, x => x);
  }
  write(key, value) {
    return this.paxos.changeQuery(key, x => value, x => x);
  }
}
```

A key/value storage with compare-and-set support may look like:

```javascript
class CASKeyValue {
  constuctor(paxos) {
    this.paxos = paxos;
  }
  read(key) {
    return this.paxos.changeQuery(key, x => x==null ? { ver: 0, val: null}, x => x);
  }
  write(key, ver, val) {
    return this.paxos.changeQuery(key, x => {
      if (x.ver != ver) throw new Error();
      return { ver: ver+1, val: val };
    }, x => x);
  }
}
```

# Consistency

A consistent system is one that does not contain a contradiction. Contradictions happen when a system breaks its
promises. Different storages provide different promises so there are different types of consistency:
eventual, weak, causal, strong and others.

Gryadka supports linearizability (a promise) so it is a strongly consistent data storage (of course if it holds 
its promise).

Intuitively, linearizability is very close to thread safety: a system of a linearizable key/value storage and its
clients behaves the same way as a thread safe hashtable and a set of threads working with it. The only difference is
that a linearizable key/value storage usually is replicated and tolerates network issues and node's crushes without
violating the consistency guarantees.

It isn't easy to keep promises, for example Kyle Kingsbury demonstrates in [his research](https://aphyr.com/tags/jepsen) 
that many commercial data storages had consistency issues, among them were: 
[VoltDB](https://aphyr.com/posts/331-jepsen-voltdb-6-3), 
[Cassandra](https://aphyr.com/posts/294-jepsen-cassandra),
[MongoDB](https://aphyr.com/posts/322-jepsen-mongodb-stale-reads) and others.

This is the reason why Gryadka was built with consistency testing in mind. **Its code to test ratio is 1:5** so
for 500 lines of Paxos there are 2500 lines of tests.

## Theory

Tests can prove that a program has errors but they can't guarantee correctness. The way to go is to write a program
based on a validated model. One can use a formal specification language like TLA+ to describe a model 
and then check it with a model checker, alternatively an algorithm (a model) can be proved by hand using logic, 
induction and other math arsenal.

Gryadka uses Single-decree Paxos (Synod) to implement a rewritable register. A write once variant of Synod is 
proved in [Paxos Made Simple](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf) paper.
The rewritable variant is its extension, I bet there is a paper describing it but I failed to find it so I 
practiced logic and proved it in this [post](http://rystsov.info/2015/09/16/how-paxos-works.html).

## Cluster membership change

The [proof easily extends](http://rystsov.info/2015/12/30/read-write-quorums.html) to support read and write quorums 
of different size which is consistent with the result of 
[Flexible Paxos: Quorum intersection revisited](https://arxiv.org/abs/1608.06696). This idea can be 
combined with [Raft's joint consensus](https://raft.github.io/slides/raftuserstudy2013.pdf) to demonstrate that 
a [simple sequence of steps changes the size of a cluster](http://rystsov.info/2016/01/05/raft-paxos.html) without violation of
consistency.

## Simulated network, mocked Redis, fault injections

Testing is done by mocking the network layer and checking consistency invariants during various 
network fault injections such as message dropping and message reordering.

Each test scenario uses seed-able randomization so all test's random decisions are determined by 
its initial value (seed) and user can replay any test and expect the same outcome.

### Invariants

The following situation is one of the examples of a consistency violation:

1. Alice reads a value
2. Alice tells Bob the observed value via an out of the system channel (a rumor)
3. Bob reads a value but the system returns a value which is older than the rumor 

It gives a hint how to check linearizability:

* Tests check a system similar to CASKeyValue
* All clients are homogeneous and execute the following loop
  1. Read a value
  2. Change it
  3. Write it back
* Clients run in the same process concurrently and spread rumors instantly after each read or write operation
* Once a client observed a value (through the read or write operation) she checks that it's equal or newer than the one known through rumors on the moment the operation started

This procedure already helped to find a couple of consistency bugs so it works :)  

In order to avoid a degradation of the consistency test to `return true;` there is the `losing/c2p2k1.i` test which
tests the consistency check on an a priory inconsistent Paxos configuration (three acceptors with quorums of size 1).  

### How to run consistency tests:

Prerequisites: node-nightly installed globally, see https://www.npmjs.com/package/node-nightly for details

1. Clone this repo
2. cd gryadka
3. npm install
4. ./bin/run-consistency-tests.sh all void seed1

Instead of "void" one can use "record" to record all events fired in the system during a simulation after it. Another
alternative is "replay" - it executes the tests and compares current events with previously written events (it was
useful to check determinism of a simulation).

It takes time to execute all test cases so run-consistency-tests.sh also supports execution of a particular test case: just
replace "all" with the test's name. Run ./bin/run-consistency-tests.sh without arguments to see which tests are supported.

#### Test scenarios

Tests are located in the `tests/consistency/scenarios` folder grouped into 4 scenarios. All the tests follow the same naming convention so if you see a test named, say, `c2p2k1` you can be sure that it checks consistency when there are two proposers and two concurrent clients (c2) working with the same key (k1). 

##### Shuffling

Shuffling tests check consistency in the presence of arbitrary network delays testing the cases when an order of the received messages does't match the order of the sent messages.

##### Losing

Losing tests deal with arbitrary network delays and arbitrary message loss.

##### Partitioning

Partitioning tests are about arbitrary network delays, arbitrary message loss and continuous connectivity issues between proposers and acceptors. 

##### Membership

Membership tests check that the consistency holds in the presence of arbitrary network delays and arbitrary message loss during the progress of extending/shrinking a cluster.

|Name | Description|
|---|---|
|c2p2k2.a3.a4 | tests a process of migration from 3 acceptors to 4 acceptors |
|c2p2k2.flux | tests a process of continuous extending/shrinking a cluster between 3 and 4 acceptors |

# FAQ

#### How does it differ from Redis Cluster?

[Redis Cluster](https://redis.io/topics/cluster-spec) is responsible for sharding and replication while Gryadka pushs 
sharding to an above layer and focuses only on replication.

Both systems execute a request only when a node executing the request is "connected" to the 
majority of the cluster. The difference is in the latency/safety trade-off decisions.

Even in the best case Gryadka requires two consequent round-trips: one is between a client and a proposer, another is 
between the proposer and the acceptors. Meanwhile Redis uses asynchronous replication and requires just one round trip: 
between a client and the master.

But the better latency has its price, [Redis's docs](https://redis.io/topics/cluster-tutorial) warns us: 
"Redis Cluster is not able to guarantee strong consistency. In practical terms this means that under certain 
conditions it is possible that Redis Cluster will lose writes that were acknowledged by the system to the client.".

Gryadka uses Paxos based master-master replication so lost writes and other consistency issues are impossible by 
design.

#### Does it support storages other than Redis?

No, the size of a pluggable storage system would have been of the same magnitude as the Gryadka's current
Paxos implementation so only Redis is supported.

The good news is that the size of the code is tiny therefore it should be easy to read, to understand and 
to rewriteit for any storage of your choice.

#### I heard that Raft is simpler than Paxos, why don't you use it?

Raft is a protocol for building replicated consistent persistent append-only log. Paxos has several flavors. 
Multi-decree Paxos does the same as Raft (log), but Single-decree Paxos replicates atomic variable.

Yes, Raft looks simpler than Multi-decree Paxos, but Single-decree Paxos is simpler than Raft because
with Paxos all the updates happen in-place and you don't need to implement log truncation and snapshotting.

Of course replicated log is a more powerful data structure than replicated variable, but for a lot of cases it's 
enough the latter. For example, a key-value storage can be build just with a set of replicated variables.

#### Why did you choose JavaScript and Redis?

Gryadka is an educational project so I chose 
[the most popular language](https://github.com/blog/2047-language-trends-on-github) on GitHub and 
[the most popular key/value storage](http://db-engines.com/en/ranking/key-value+store) (according to db-engines).