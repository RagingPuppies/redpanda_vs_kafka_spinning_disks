# Benchmarking Redpanda vs Apache Kafka on spinning disks
So after googling a bunch of articles in google and getting some information I haven't found a valid benchmark.
We've decided to make a few tests to understand if this is a path we want to take.
In my company, we are not using k8s for stateful apps (not yet) and we try to utlize the full power of the bare-metal.

### The Setup
For the first run, we will compare one Kafka node versus one Redpanda node, no clusters, quroms and such.
Each node gets the following: 
- 24 CPUs
- 64GB RAM
- 3 Data Disks - spinning, with 7200RPM
- Ubuntu 20.

### The Tools
I will be using the `rpk` for Redpanda and the kafka `/bin` scripts for Kafka, 
I will use `kafka-producer-perf-test` with kafka client 3.2.3 to produce the load X6 threads.


## Kafka Configurations
I will use the following relevant configurations, from what i've tested, they seemed to allow the heighest throughput.
```
num.io.threads=16
socket.request.max.bytes=104857600
socket.send.buffer.bytes=104857600
socket.receive.buffer.bytes=104857600
log.segment.bytes=536870912
log.retention.check.interval.ms=60000
num.network.threads=8
num.replica.fetchers=4
group.max.session.timeout.ms=300000
log.cleaner.enable=true
log.cleaner.threads=2
num.recovery.threads.per.data.dir=4
log.dirs=/Prod/kafka/c/data,/Prod/kafka/d/data,/Prod/kafka/e/data,/Prod/kafka/f/data
log.retention.hours=1
log.retention.bytes=-1
```

I've created a topic with 10 partitions (more than that seems to impact performance).
```
./kafka-topics.sh --bootstrap-server localhost:9092 --create --topic rp_lt --replica-assignment 40001,40001,40001,40001,40001,40001,40001,40001,40001,40001
```

## Redpanda Configurations
First, It seemed that running `rpk iotune` which supposed to learn my system I/O capcity and create some sort of thresholds didn't worked correctly.
So when running the service, I've excluded: `--io-properties-file=/etc/redpanda/io-config.yaml` another option is to edit the file and set am higher value.

Second, I've used `--unsafe-bypass-fsync=true` this is removing the promise that the record actually was written to the disk, so in case of disaster, data might be lost.
Redpanda team are not in favor of this setting, but TBH, i don't think that kafka is flushing it's memory when it sends acks. so once you work with replication factor higher than 1, and set ack to all in a cluster, it should be sort of ok.

At this point i've started the service manually:
```
/opt/redpanda/bin/redpanda --redpanda-cfg /etc/redpanda/redpanda.yaml --lock-memory=false --unsafe-bypass-fsync=true
```

and i've also created a topic with 10 partitions:
```
rpk topic create rp_lt -p 10
TOPIC  STATUS
rp_lt  OK
```


## Results


| System       | Records/s | Throughput | avg Latency | Total Produced |
|--------------|-----------|------------|-------------|----------------|
| Apache Kafka | 235,000   | 222MBps    | 250ms       | 6,000,000      |
| Redpanda     | 204,000   | 198MBps    | 602ms       | 6,000,000      |
