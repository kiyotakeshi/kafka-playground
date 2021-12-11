# kafka playground

## prepare topic

```shell
docker compose up -d

docker compose ps

# create topic and partitions
$ docker compose exec -T kafka1 /bin/bash <<EOF
/opt/bitnami/kafka/bin/kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --replication-factor 1 --partitions 4
EOF

# describe topic
$ docker compose exec -T kafka1 /bin/bash <<EOF
/opt/bitnami/kafka/bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test-topic
EOF
```

## test message without key

```shell
# consumer
$ docker compose exec kafka1 bash

$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic

# producer
$ docker compose exec kafka1 bash

$ /opt/bitnami/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic
# >test
# >hello
# >goodbye
# >1
# >2
# >3

$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic test-topic --from-beginning -property "key.separator= - " --property "print.key=true"
# null - hello
# null - 1
# null - 2

# consumer
$ docker compose exec kafka3 bash

# display in no particular order(random)
$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic --from-beginning
```

## prepare topic

```shell
$ docker compose exec kafka1 bash

# create topic and partitions
$ /opt/bitnami/kafka/bin/kafka-topics.sh --create --topic test-topic-with-key \
--bootstrap-server localhost:9092 --replication-factor 1 --partitions 4

# describe topic
$ /opt/bitnami/kafka/bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic test-topic-with-key
```

## test message with key

```shell
# producer
$ /opt/bitnami/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 \
--topic test-topic-with-key --property "key.separator=-" --property "parse.key=true"
# >A-Apple
# >A-Anger
# >B-Bear
# >C-Cow

# consumer
# display in no particular order(random)
$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic test-topic-with-key --from-beginning -property "key.separator= - " --property "print.key=true"
# B - Bear
# A - Anger
# C - Cow
# A - Apple
```

## check offset

```shell
$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
# __consumer_offsets
# test-topic
# test-topic-with-key
```

## consumer group

```shell
$ docker compose exec kafka1 bash

$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test-topic-with-key --from-beginning

# consumer group ID is mandatory and automatically generated
$ /opt/bitnami/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
# console-consumer-28359
# console-consumer-99389

# instantiate a console consumer with specified id
$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic test-topic --group console-consumer
```

```shell
$ docker compose exec kafka2 bash

# same consumer group
$ /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
--topic test-topic --group console-consumer
```

### send message

```shell
# whichever you want (kafka1,2,3)
$ docker compose exec kafka3 bash

# check consumer group
$ /opt/bitnami/kafka/bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
# console-consumer

# create record
# display two console consumer one after another(alternately) approximately
$ /opt/bitnami/kafka/bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test-topic
# >1
# >2
# >3
# >4
```

## replica in 3 brokers cluster

```shell
$ docker compose down --remove-orphans

$ docker compose up -d

$ docker compose exec kafka1 bash

# In this pattern is pattern, we can't read message from down broker's partition
$ /opt/bitnami/kafka/bin/kafka-topics.sh --create --topic test-topic \
--bootstrap-server localhost:9092 --replication-factor 1 --partitions 4

$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic
# Topic: test-topic	TopicId: l98J9-heSYqUL3nCKMtYNQ	PartitionCount: 4	ReplicationFactor: 1	Configs: segment.bytes=1073741824
# 	Topic: test-topic	Partition: 0	Leader: 1003	Replicas: 1003	Isr: 1003
# 	Topic: test-topic	Partition: 1	Leader: 1002	Replicas: 1002	Isr: 1002
# 	Topic: test-topic	Partition: 2	Leader: 1001	Replicas: 1001	Isr: 1001
# 	Topic: test-topic	Partition: 3	Leader: 1003	Replicas: 1003	Isr: 1003

# create topic and partitions with multiple replication
$ /opt/bitnami/kafka/bin/kafka-topics.sh --create --topic test-topic-replicated \
--bootstrap-server localhost:9092 --replication-factor 3 --partitions 4

# describe topic
$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic-replicated
# Topic: test-topic-replicated	TopicId: f4TSWAn_RoGDhacY24BaQA	PartitionCount: 4	ReplicationFactor: 3	Configs: segment.bytes=1073741824
# 	Topic: test-topic-replicated	Partition: 0	Leader: 1003	Replicas: 1003,1002,1001	Isr: 1003,1002,1001
# 	Topic: test-topic-replicated	Partition: 1	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1001,1003
# 	Topic: test-topic-replicated	Partition: 2	Leader: 1001	Replicas: 1001,1003,1002	Isr: 1001,1003,1002
# 	Topic: test-topic-replicated	Partition: 3	Leader: 1003	Replicas: 1003,1001,1002	Isr: 1003,1001,1002
```

```shell
# down one broker
$ docker-compose stop kafka2

# in kafka1 container(broker)
# partition leader and Isr(In-Sync Replica) info are changed
$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic-replicated
# Topic: test-topic-replicated	TopicId: f4TSWAn_RoGDhacY24BaQA	PartitionCount: 4	ReplicationFactor: 3	Configs: segment.bytes=1073741824
# 	Topic: test-topic-replicated	Partition: 0	Leader: 1003	Replicas: 1003,1002,1001	Isr: 1003,1002
# 	Topic: test-topic-replicated	Partition: 1	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1003
# 	Topic: test-topic-replicated	Partition: 2	Leader: 1003	Replicas: 1001,1003,1002	Isr: 1003,1002
# 	Topic: test-topic-replicated	Partition: 3	Leader: 1003	Replicas: 1003,1001,1002	Isr: 1003,1002

$ docker-compose start kafka2

# Isr(In-Sync Replica) info is changed(partition leader keeps)
$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic-replicated
# Topic: test-topic-replicated	TopicId: f4TSWAn_RoGDhacY24BaQA	PartitionCount: 4	ReplicationFactor: 3	Configs: segment.bytes=1073741824
# 	Topic: test-topic-replicated	Partition: 0	Leader: 1003	Replicas: 1003,1002,1001	Isr: 1003,1002,1001
# 	Topic: test-topic-replicated	Partition: 1	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1003,1001
# 	Topic: test-topic-replicated	Partition: 2	Leader: 1003	Replicas: 1001,1003,1002	Isr: 1003,1002,1001
# 	Topic: test-topic-replicated	Partition: 3	Leader: 1003	Replicas: 1003,1001,1002	Isr: 1003,1002,1001

$ docker-compose stop kafka3

# partition leader and Isr(In-Sync Replica) info are changed
$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic-replicated
# Topic: test-topic-replicated	TopicId: f4TSWAn_RoGDhacY24BaQA	PartitionCount: 4	ReplicationFactor: 3	Configs: segment.bytes=1073741824
# 	Topic: test-topic-replicated	Partition: 0	Leader: 1002	Replicas: 1003,1002,1001	Isr: 1002,1001
# 	Topic: test-topic-replicated	Partition: 1	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1001
# 	Topic: test-topic-replicated	Partition: 2	Leader: 1001	Replicas: 1001,1003,1002	Isr: 1002,1001
# 	Topic: test-topic-replicated	Partition: 3	Leader: 1001	Replicas: 1003,1001,1002	Isr: 1002,1001

$ docker-compose start kafka3

$ /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test-topic-replicated
# Topic: test-topic-replicated	TopicId: f4TSWAn_RoGDhacY24BaQA	PartitionCount: 4	ReplicationFactor: 3	Configs: segment.bytes=1073741824
# 	Topic: test-topic-replicated	Partition: 0	Leader: 1002	Replicas: 1003,1002,1001	Isr: 1002,1001,1003
# 	Topic: test-topic-replicated	Partition: 1	Leader: 1002	Replicas: 1002,1001,1003	Isr: 1002,1001,1003
# 	Topic: test-topic-replicated	Partition: 2	Leader: 1001	Replicas: 1001,1003,1002	Isr: 1002,1001,1003
# 	Topic: test-topic-replicated	Partition: 3	Leader: 1001	Replicas: 1003,1001,1002	Isr: 1002,1001,1003
```

### broker 1

```shell
# share the 4 partition in 3 broker
$ \ls .docker/kafka1/data/test-topic-?
# 00000000000000000000.index      00000000000000000000.log        00000000000000000000.timeindex  leader-epoch-checkpoint         partition.metadata

# each broker has all partition
$ \ls -ld .docker/kafka1/data/test-topic-replicated-*
# drwxr-xr-x  7 kiyotakeshi  staff  224 Dec 11 18:48 .docker/kafka1/data/test-topic-replicated-0
# drwxr-xr-x  7 kiyotakeshi  staff  224 Dec 11 18:48 .docker/kafka1/data/test-topic-replicated-1
# drwxr-xr-x  7 kiyotakeshi  staff  224 Dec 11 18:48 .docker/kafka1/data/test-topic-replicated-2
# drwxr-xr-x  7 kiyotakeshi  staff  224 Dec 11 18:48 .docker/kafka1/data/test-topic-replicated-3
```

### broker 2

```shell
# share the 4 partition in 3 broker
$ file .docker/kafka2/data/test-topic-?
# .docker/kafka2/data/test-topic-0: directory
# .docker/kafka2/data/test-topic-3: directory

# each broker has all partition
$ file .docker/kafka2/data/test-topic-replicated-*
# .docker/kafka2/data/test-topic-replicated-0: directory
# .docker/kafka2/data/test-topic-replicated-1: directory
# .docker/kafka2/data/test-topic-replicated-2: directory
# .docker/kafka2/data/test-topic-replicated-3: directory
```

### broker 3

```shell
# share the 4 partition in 3 broker
$ file .docker/kafka3/data/test-topic-?
# .docker/kafka3/data/test-topic-2: directory

# each broker has all partition
$ file .docker/kafka3/data/test-topic-replicated-*
# .docker/kafka3/data/test-topic-replicated-0: directory
# .docker/kafka3/data/test-topic-replicated-1: directory
# .docker/kafka3/data/test-topic-replicated-2: directory
# .docker/kafka3/data/test-topic-replicated-3: directory
```
