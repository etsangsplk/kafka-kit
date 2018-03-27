## Overview

Topicmappr was created as a replacement for Kafka's provided `kafka-reassign-partition.sh` tool, providing additional enhancements:

### Deterministic Output
Given the same input, topicmappr will always provide the same output map.

### Minimal Partition Movement
Avoids reassigning partitions where movement isn't necessary, greatly reducing reassignment times and resource load for simple recoveries.

### Balancing Partition Placement With Constraints
For each broker pending replacement, topicmappr chooses the least-utilized candidate broker (based on partition counts or storage available, configurable via the `-placement` param) that satisfies the following constraints:

- the broker isn't already in the replica set
- the broker isn't in any of the existing replica set localities (looks up the built in `rack-id` parameter)

Provided enough brokers, topicmapper determines the appropriate leadership, follower and failure domain balance.

### Summary Output
An output of what's changed along with advisory notices (e.g. insufficient broker counts supplied to satisfy all constraints at the desired partition/replica count).

## Usage
Flags:

```
Usage of topicmappr:
  -brokers string
    	Broker list to rebuild topic partition map with
  -force-rebuild
    	Forces a rebuild even if all existing brokers are provided
  -ignore-warns
    	Whether a map should be produced if warnings are emitted
  -out-file string
    	If defined, write a combined map of all topics to a file
  -out-path string
    	Path to write output map files to
  -placement string
    	Partition placement type: [count, storage] (default "count")
  -rebuild-map string
    	Rebuild a topic map
  -rebuild-topics string
    	Rebuild topics (comma delim list) by lookup in ZooKeeper
  -replication int
    	Change the replication factor
  -use-meta
    	Use broker metadata as constraints (default true)
  -zk-addr string
    	ZooKeeper connect string (for broker metadata or rebuild-topic lookups) (default "localhost:2181")
  -zk-metrics-prefix string
    	ZooKeeper namespace prefix (for Kafka metrics) (default "topicmappr")
  -zk-prefix string
    	ZooKeeper namespace prefix (for Kafka)
```

### How Mapping Works

Topicmappr primarily takes two inputs: a topic map (either looked up in ZooKeeper with `--rebuild-topics` or provided literally with `--rebuild-map`) and a list of brokers. Topicmappr builds a map that ensures the referenced topics are mapped to the listed brokers.

Subsequent executions should always produce the same output map (\*provided the same input; a change in topic state in ZooKeeper yields a different input). In order to uphold the intent of "minimal partition movement", topicmappr only maps a partition to a new broker if an existing broker isn't provided in the broker list.

For instance, if `test_topic` were initially built on brokers `1001,1002`, a run with `1001,1002,1003,1004` would be a no-op. An example that can be tested locally using a mock map:


> $ topicmappr -rebuild-map '{"version":1,"partitions":[{"topic":"test_topic","partition":0,"replicas":[1001,1002]},{"topic":"test_topic","partition":1,"replicas":[1002,1001]},{"topic":"test_topic","partition":2,"replicas":[1001,1002]},{"topic":"test_topic","partition":3,"replicas":[1002,1001]}]}' -brokers=1001,1002,1003,1004 -use-meta=false

```
Topics:
  test_topic

Broker change summary:
  Replacing 0, added 0, total count changed by 0

Action:
  no-op

WARN:
  [none]

Partition map changes:
  test_topic p0: [1001 1002] -> [1001 1002]
  test_topic p1: [1002 1001] -> [1002 1001]
  test_topic p2: [1001 1002] -> [1001 1002]
  test_topic p3: [1002 1001] -> [1002 1001]
...
```

If broker `1002` failed, we would exclude it from the list, indicating that those positions be filled with the best fitting broker in the list:

> $ topicmappr -rebuild-map '{"version":1,"partitions":[{"topic":"test_topic","partition":0,"replicas":[1001,1002]},{"topic":"test_topic","partition":1,"replicas":[1002,1001]},{"topic":"test_topic","partition":2,"replicas":[1001,1002]},{"topic":"test_topic","partition":3,"replicas":[1002,1001]}]}' -brokers=1001,1003,1004 -use-meta=false

```
Topics:
  test_topic

Broker change summary:
  Broker 1002 marked for removal
  Replacing 1, added 2, total count changed by 1

Action:
  Rebuild topic with 1 broker(s) marked for removal

WARN:
  [none]

Partition map changes:
  test_topic p0: [1001 1002] -> [1001 1003] replaced broker
  test_topic p1: [1002 1001] -> [1003 1001] replaced broker
  test_topic p2: [1001 1002] -> [1001 1004] replaced broker
  test_topic p3: [1002 1001] -> [1004 1001] replaced broker

Partitions assigned:
  Broker 1001 - leader: 2, follower: 2, total: 4
  Broker 1003 - leader: 1, follower: 1, total: 2
  Broker 1004 - leader: 1, follower: 1, total: 2
...
```

Essentially, if all brokers handling a topic are a subset of the provided list, nothing changes. The default action is to only fix what's broken.

This can be overridden with the `--force-rebuild` option, tells topicmappr to rebuild an ideal map from the broker list, disregarding the state of the topic:

> $ topicmappr -rebuild-map '{"version":1,"partitions":[{"topic":"test_topic","partition":0,"replicas":[1001,1002]},{"topic":"test_topic","partition":1,"replicas":[1002,1001]},{"topic":"test_topic","partition":2,"replicas":[1001,1002]},{"topic":"test_topic","partition":3,"replicas":[1002,1001]}]}' -brokers=1001,1002,1003,1004 -use-meta=false --force-rebuild

```
Topics:
  test_topic

Broker change summary:
  Replacing 0, added 2, total count changed by 2

Action:
  Expanding/rebalancing topic with 2 broker(s) (this is a no-op unless --force-rebuild is specified)

WARN:
  [none]

Partition map changes:
  test_topic p0: [1001 1002] -> [1001 1002]
  test_topic p1: [1002 1001] -> [1002 1001]
  test_topic p2: [1001 1002] -> [1003 1004] replaced broker
  test_topic p3: [1002 1001] -> [1004 1003] replaced broker

Partitions assigned:
  Broker 1001 - leader: 1, follower: 1, total: 2
  Broker 1002 - leader: 1, follower: 1, total: 2
  Broker 1003 - leader: 1, follower: 1, total: 2
  Broker 1004 - leader: 1, follower: 1, total: 2
...
```

#### rebuild-topics
Takes a comma delimited list of topic names and a list of target brokers. The broker map is fetched from ZooKeeper and rebuilt with the supplied broker list.

Topic name lookup with regex is supported\*. For instance, providing `--rebuild-topics="test_topic[0-9]"` might return `test_topic1`, `test_topic2`, and `test_topic3`. Name literals and regex can be combined: `--rebuild-topics="test_topic,numbered_topics[0-9]"`.

> $ topicmappr -rebuild-topics test_topic -brokers "0,2" -zk-addr "localhost:2181"

```
Topics:
  test_topic

Broker change summary:
  Broker 1 marked for removal
  Replacing 1, added 1, total count changed by 0

Action:
  Rebuild topic with 1 broker(s) marked for removal

WARN:
  [none]

Partition map changes:
  test_topic p0: [0 1] -> [0 2] replaced broker
  test_topic p1: [1 0] -> [2 0] replaced broker
  test_topic p2: [0 1] -> [0 2] replaced broker
  test_topic p3: [1 0] -> [2 0] replaced broker
  test_topic p4: [0 1] -> [0 2] replaced broker
  test_topic p5: [1 0] -> [2 0] replaced broker
  test_topic p6: [0 1] -> [0 2] replaced broker
  test_topic p7: [1 0] -> [2 0] replaced broker

Partitions assigned:
  Broker 0 - leader: 4, follower: 4, total: 8
  Broker 2 - leader: 4, follower: 4, total: 8

New parition maps:
  test_topic.json
```

\*Take note of how regex is interpreted: all topics included in the `--rebuild-topics` list ultimately become regex. Topic names where all characters are those allowed by Kafka (`a-z`, `A-Z`, `0-9`, `_`, `-`, `.`) sans `.`, are assumed to be literal names and thus become the regex `/^topic$/`. The inclusion of a `.` or any other character assumes that the entry is to be interpreted as regex and is compiled as is. This means that if you want to rebuild the literal topic `my.topic`, it's best to provide `--rebuild-topics="my\.topic"`. Without escaping the `.` (`--rebuild-topics="my.topic"`), both `my.topic` and `my1topic` would be targeted.

#### rebuild-map
Takes an existing topic map and a list of target brokers. A topic initially built with the brokers `[1001,1002,1003]` that lost broker `1003` could be rebuilt by supplying the new broker list `[1001,1002,1004]`.

Multiple topics can be included in the topic map.

Example:

> $ topicmappr -rebuild-map '{"version":1,"partitions":[{"topic":"test_topic","partition":0,"replicas":[1005,1006]},{"topic":"test_topic","partition":2,"replicas":[1007,1001]},{"topic":"test_topic","partition":7,"replicas":[1007,1002]},{"topic":"test_topic","partition":6,"replicas":[1006,1001]},{"topic":"test_topic","partition":4,"replicas":[1002,1005]},{"topic":"test_topic","partition":5,"replicas":[1005,1007]},{"topic":"test_topic","partition":3,"replicas":[1001,1002]},{"topic":"test_topic","partition":1,"replicas":[1006,1007]}]}' -brokers="1001,1002,1003,1004,1005,1006,1008" -use-meta=false

```
Topics:
  test_topic

Broker change summary:
  Broker 1007 marked for removal
  Replacing 1, added 3, total count changed by 2

Action:
  Rebuild topic with 1 broker(s) marked for removal

WARN:
  [none]

Partition map changes:
  test_topic p0: [1005 1006] -> [1005 1006]
  test_topic p1: [1006 1007] -> [1006 1003] replaced broker
  test_topic p2: [1007 1001] -> [1003 1001] replaced broker
  test_topic p3: [1001 1002] -> [1001 1002]
  test_topic p4: [1002 1005] -> [1002 1005]
  test_topic p5: [1005 1007] -> [1005 1008] replaced broker
  test_topic p6: [1006 1001] -> [1006 1001]
  test_topic p7: [1007 1002] -> [1004 1002] replaced broker

Partitions assigned:
  Broker 1002 - leader: 1, follower: 2, total: 3
  Broker 1008 - leader: 0, follower: 1, total: 1
  Broker 1004 - leader: 1, follower: 0, total: 1
  Broker 1005 - leader: 2, follower: 1, total: 3
  Broker 1006 - leader: 2, follower: 1, total: 3
  Broker 1003 - leader: 1, follower: 1, total: 2
  Broker 1001 - leader: 1, follower: 2, total: 3

New parition maps:
  test_topic.json
```

### Placement Strategy

The `-placement` parameter takes one of two values: `count` or `storage`. This determines how brokers are chosen in partition placement.

#### Count

The count strategy balances partitions in a way that results in the most even number across brokers. This is simple and reliable if imbalances in data volumes among partitions is not anticipated.

#### Storage

The storage strategy chooses brokers based on free space. In each placement decision, the broker with the most available free space that satisfies all other constraints is chosen. The storage strategy is best used if large imbalances among partitions is anticipated.

When using the storage placement strategy, an estimate of changes in free storage is printed in the topicmappr summary output:

```
Partitions assigned:
  Broker 1006 - leader: 9, follower: 12, total: 21
  Broker 1003 - leader: 9, follower: 8, total: 17
  Broker 1005 - leader: 12, follower: 11, total: 23
  Broker 1007 - leader: 12, follower: 11, total: 23
  Broker 1002 - leader: 13, follower: 10, total: 23
  Broker 1004 - leader: 9, follower: 12, total: 21

Storage free change estimations:
  Broker 1006 - 1019.21 -> 1111.45 (+92.25GB, 9.05%)
  Broker 1004 - 1067.73 -> 1102.47 (+34.74GB, 3.25%)
  Broker 1005 - 1108.10 -> 1099.70 (-8.40GB, -0.76%)
  Broker 1002 - 1173.95 -> 1088.83 (-85.13GB, -7.25%)
  Broker 1007 - 1194.80 -> 1079.82 (-114.98GB, -9.62%)
  Broker 1003 - 1020.79 -> 1102.31 (+81.52GB, 7.99%)
```

The storage strategy requires complete metrics data in order to operate. Topicmappr will check for the following znodes as children of `/topicmappr` (configurable via `-zk-metrics-prefix`):

**/topicmappr/partitionmeta**

The znode data must be formatted as JSON with the structure `{"<topic name>": {"<partition number>": {"Size": <bytes>}}}`. Metrics data for all topics being mapped must be present for all partitions.

**/topicmappr/brokermetrics**

The znode data must be formatted as JSON with the structure `{"<broker ID>": {"StorageFree": <bytes>}}`. Metrics data for all brokers participating brokers in a mapping operation must be present.

This data can be populated from any metrics system as long as it conforms to these standards. A tool that populates this data using Datadog metrics is being developed.

### Scaling and Managing Topics

#### Up

Based on the mapping mechanics described in the "How Mapping Works" section, topicmappr provides two ways to scale a topic over more brokers.

Full rebuild: Provide topicmappr a completely new list of brokers (e.g. none of the provided brokers are any of those already hosting the topic). The disadvantage of this method is that 100% of the data must be moved. The advantage is a reduced risk of running a broker out of storage (compared to an in place scaling) and more total bandwidth will be available for the resize (original+new brokers).

In place scaling: Provide topicmappr with a broker list that includes those already hosting the topic plus additional brokers. Run topicmappr with `--force-rebuild`. The potential advantage of this is possibly reduced data movement fewer new brokers than a full rebuild scale up.

#### Down

Topics can also be easily scaled down. Simply trim the desired of brokers from the broker list and topicmappr will rebalance the partitions over the remaining brokers.

## Safeties Using ZooKeeper State
Topicmappr references cluster state from ZooKeeper, automatically enforcing the following:

- The `broker.rack` Kafka attribute is used as a placement constraint. No replica set shall have brokers that share this attribute.
- All provided brokers exist and are visible in ZooKeeper.
- The topic exists in ZooKeeper.

Example attempting to rebuild a topic with a non-existent broker:

> $ topicmappr -rebuild-topic test_topic -brokers "0,3"

```
Broker change summary:
  Broker 1 marked for removal
  Broker 3 not found in ZooKeeper
  Replacing 1, added 0, total count changed by -1

Action:
  Shrinking topic by 1 broker(s)

WARN:
  test_topic p0: No additional brokers that meet constraints
  test_topic p1: No additional brokers that meet constraints
  test_topic p2: No additional brokers that meet constraints
  test_topic p3: No additional brokers that meet constraints
  test_topic p4: No additional brokers that meet constraints
  test_topic p5: No additional brokers that meet constraints
  test_topic p6: No additional brokers that meet constraints
  test_topic p7: No additional brokers that meet constraints

Partition map changes:
  test_topic p0: [0 1] -> [0] replaced broker
  test_topic p1: [1 0] -> [0] replaced broker
  test_topic p2: [0 1] -> [0] replaced broker
<...>
```