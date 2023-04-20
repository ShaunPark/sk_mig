본 가이드에서는 Kafka Mirror Maker 에서 Confluent Replicator로 Migration 하는 방법을 다룹니다. 

- **Confluent Platform Version: 7.3.2**

---

---

# 1. Mirror Maker 1 to Replicator

### 1-1. Test Topic 생성

- source cluster 서버

```bash
$ /engn/dc1/confluent-7.3.2/bin/kafka-topics --bootstrap-server br01:9092 --create --topic london
```

### 1-2. Mirror Maker 1 실행

- target cluster 서버

```bash
### mirror-maker 설정파일 구성 
$ vi /engn/dc2/etc/kafka/mm1_consumer.properties

bootstrap.servers=br01:9092
exclude.internal.topics=true
client.id=mm1_consumer_client01
group.id=mm1_consumer_group01

$ vi /engn/dc2/etc/kafka/mm1_producer.properties

bootstrap.servers=sr-ksql01:9092
client.id=mirror_maker_producer-01

### mirror-maker 실행
#!/bin/bash
/engn/dc2/confluent-7.3.2/bin/kafka-mirror-maker \
--producer.config /engn/dc2/etc/kafka/mm1_producer.properties \
--consumer.config /engn/dc2/etc/kafka/mm1_consumer.properties \
--num.streams 1 --whitelist rome
```

### 1-3. Test Topic 부하

- source cluster 서버

```bash
$ /engn/dc1/confluent-7.3.2/bin/kafka-producer-perf-test  --num-records 10000 --throughput 10 --record-size 1024 --producer-props bootstrap.servers=br01:9092 --topic rome
```

- mirroring test

```bash
### source topic 

$ /engn/dc1/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server br01:9092 --to
pic rome --time -1
rome:0:14
rome:1:16
rome:2:0

### target topic

$ /engn/dc2/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server sr-ksql01:9092 --topic rome --time -1
rome:0:0
rome:1:30
rome:2:0
```

### 1-4. Mirror Maker 1  >> Replicator

Mirror Maker에 이어서 동일한 토픽을 Replicator로 가져오기 위해 consumer group id 를 동일하게 설정한다.

`src.consumer.group.id` in Replicator must match `group.id` in MirrorMaker

- `group.id` in MirrorMaker1확인

```bash
$ /engn/dc1/confluent-7.3.2/bin/kafka-consumer-groups --bootstrap-server br01:9092 --list --all-groups
ConfluentTelemetryReporterSampler--6161840062404504535
**mm1_consumer_group01**
```

- `src.consumer.group.id` in Replicator 설정 준비

```bash
### replicator_consumer.properties
bootstrap.servers=br01:9092
group.id=**mm1_consumer_group01**
topic.preserve.partitions=true

### replicator_producer.properties
bootstrap.servers=sr-ksql01:9092

### quickstart_replicator.properties
confluent.topic.replication.factor=1
offset.storage.replication.factor=1
config.storage.replication.factor=1
status.storage.replication.factor=1

topic.rename.format=${topic}
```

- target cluster 서버 : mirror maker 1 down

```bash
$ kill <mirror_maker_process_id>
```

- target cluster 서버: replicator 시작

```bash
### start-replicator.sh
#!/bin/bash
/engn/dc2/confluent-7.3.2/bin/replicator --cluster.id replicator_cluster \
--producer.config /engn/dc2/etc/kafka/replicator_producer.properties \
--consumer.config /engn/dc2/etc/kafka/replicator_consumer.properties \
--replication.config /engn/dc2/etc/kafka/quickstart_replicator.properties --whitelist rome
```

- Test Topic 부하 종료 후 Source/Target Cluster offset 확인

```bash
### source cluster 
$ /engn/dc1/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server br01:9092 --topic rome --time -1
rome:0:725
rome:1:589
rome:2:782
=2096

### target cluster 
$  /engn/dc2/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server sr-ksql01:9092 --topic rome --time -1
rome:0:741
rome:1:561
rome:2:794
=2096
```

<aside>
💡 **MirrorMaker v1 제한점:**
- **TOPIC 설정사항까지 복제하지 못한다:** destination cluster에서 “auto.create.topics.enable” 옵션이 “true”로 설정되어 있는 경우, source cluster에 해당 토픽의 설정사항은 고려하지 않고 단지 destination cluster의 디폴트 값에 따라 복제 topic을 생성한다. mirror maker 기동 후 Source topic 의 설정 변경이 일어나는 경우에도, 복제 토픽의 설정을 수동으로 변경해주어야 한다.
- **Message Offset 이 다를 수 있다:** Source topic 이 생성된 이후 아직 메세지가 안들어있는 상태에서 mirror maker를 기동한 경우 source/target 토픽 간 메세지 오프셋이 같을 수 있으나 Producer의 retry 에 따라 불일치가 발생할 수 있다. 따라서 Failover/Failback 시나리오에 해당 MirrorMaker1사용은 권장되지 않는다.

</aside>

# 2. Mirror Maker 2 to Replicator

 Mirror Maker 2 (이하mm2)는 Source Connector의 형식으로 동작하며 source 토픽을 target으로 복제해온다. Mirror Maker 1과 달리 Mirror Maker 2를 기동했을 때 source cluster에서는 mm2 consumer group을 확인할 수 없다. 

따라서 현재 mm2가 읽고있는 Source Topic의 offset 정보를 전용 offset topic (target cluster 에 생성된 *mm2-offsets.<source-cluster-alias>.internal 토픽*) 에 저장하며 동작하고 있다.  

```bash
/engn/dc2/confluent-7.3.2/bin/kafka-console-consumer  --bootstrap-server sr-ksql01:9092 --topic mm2-offsets.source.internal --from-beginning --property print.key=true --property key.serperator="-"
...
["MirrorSourceConnector",{"cluster":"source","partition":1,"topic":"london"}]-{"offset":89}
["MirrorSourceConnector",{"cluster":"source","partition":0,"topic":"london"}]-{"offset":125}
...
```

 

따라서 mm2 를 replicator로 마이그레이션하는 경우, consumer.group.id를 일치시키는 방법 대신 offset topic의 메세지 조작이 필요하다. 

### 2-1. Test Topic 생성

- source cluster 서버

```bash
/engn/dc1/confluent-7.3.2/bin/kafka-topics --bootstrap-server br01:9092 --create --topic london
```

### 2-2. Mirror Maker 2 실행

- target cluster 서버

```bash
### mirror-maker 설정파일 구성 
$ vi /engn/dc2/etc/kafka/connect-mirror-maker.properties

clusters = source, dest
source.bootstrap.servers = br01:9092
dest.bootstrap.servers = sr-ksql01:9092

source->dest.enabled = true
source->dest.topics = london
topics.blacklist="*.internal,__.*"
dest->source.enabled = false

replication.factor=1
checkpoints.topic.replication.factor=1
heartbeats.topic.replication.factor=1
offset-syncs.topic.replication.factor=1
offset.storage.replication.factor=1
status.storage.replication.factor=1
config.storage.replication.factor=1

sync.topic.acls.enabled = false

### mirror-maker 실행
$ /engn/dc2/confluent-7.3.2/bin/connect-mirror-maker  /engn/dc2/etc/kafka/connect-mirror-maker.properties
```

### 2-3. Test Topic 부하

- source cluster 서버

```bash
$ /engn/dc1/confluent-7.3.2/bin/kafka-producer-perf-test  --num-records 30000  --throughput 10 --record-size 1024 --producer-props bootstrap.servers=br01:9092 --topic london
```

### 2-4. Mirror Maker 2  >> Replicator

- target cluster 서버 : mirror maker 2 down

```bash
$ kill <mirror_maker_process_id>
```

- target cluster 에 복제된 topic offset 확인

```bash
$ /engn/dc2/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server sr-ksql01:9092 --topic source.london --time -1
source.london:0:**126**
source.london:1:**90**
source.london:2:**136**
```

- mm2에 이어서 replicator가 이어서 읽어올 수 있도록 replicator 의 default offset topic 생성 및 메세지 pub
    - connect-offsets 생성 : 해당 토픽이 이미 있는 경우 해당 작업은 생략한다.
        
        ```bash
        $ /engn/dc2/confluent-7.3.2/bin/kafka-topics --bootstrap-server sr-ksql01:9092 --create --replication-factor 1 --partitions 1 --topic connect-offsets --config "cleanup.policy=compact"
        ```
        
    - connect-offsets 토픽 메세지 pub 통한 replicator가 이어서 읽어올 offset 지정
        
        ```bash
        $ /engn/dc2/confluent-7.3.2/bin/kafka-console-producer --bootstrap-server sr-ksql01:9092 --topic connect-offsets --property parse.key=true --property key.separator="-"
        
        >["replicator",{"topic":"london","partition":0}]-{"offset":125}
        >["replicator",{"topic":"london","partition":1}]-{"offset":89}
        >["replicator",{"topic":"london","partition":2}]-{"offset":135}
        ```
        
        <aside>
        💡 *["replicator",{"topic":"london","partition":0}]-{"offset":125}*
        **>>> replicator 라는  group이 Source 토픽인 “london“ 의 파티션 0번을 125번 offset 까지 읽은 상태**
        
        </aside>
        
- target cluster 서버: replicator 시작

```bash
### replicator 설정 파일 
$ vi /engn/dc2/etc/kafka/replicator_producer.properties

bootstrap.servers=sr-ksql01:9092

$ vi /engn/dc2/etc/kafka/replicator_consumer.properties

bootstrap.servers=br01:9092
topic.preserve.partitions=true

$ vi  /engn/dc2/etc/kafka/quickstart_replicator.properties

confluent.topic.replication.factor=1
offset.storage.replication.factor=1
config.storage.replication.factor=1
status.storage.replication.factor=1

topic.rename.format=source.${topic}

### replicator 실행 
$ /engn/dc2/confluent-7.3.2/bin/replicator --cluster.id replicator_cluster \
--producer.config /engn/dc2/etc/kafka/replicator_producer.properties \
--consumer.config /engn/dc2/etc/kafka/replicator_consumer.properties \
--replication.config /engn/dc2/etc/kafka/quickstart_replicator.properties --whitelist london
```

- Test Topic 부하 종료 후 Source/Target Cluster offset 확인

```bash
### source cluster 
$ /engn/dc1/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server br01:9092 --topic london --time -1
london:0:1867
london:1:1922
london:2:2029

### target cluster 
$  /engn/dc2/confluent-7.3.2/bin/kafka-run-class kafka.tools.GetOffsetShell --bootstrap-server sr-ksql01:9092 --topic source.london --time -1
source.london:0:1867
source.london:1:1922
source.london:2:2029
```

<aside>
<img src="/icons/code_lightgray.svg" alt="/icons/code_lightgray.svg" width="40px" /> **위의 반복되는 작업을 실행해주는 스크립트: (target cluster에서 실행)**

```bash
#!/bin/bash

SOURCE_CLUSTER_ALIAS="source"
REPLICATOR_CONSUMER_GROUP="replicator"
REPLICATOR_OFFSET_TOPIC="connect-offsets"
BROKER_LIST="sr-ksql01:9092"
MM2_OFFSET_TOPIC="mm2-offsets."${SOURCE_CLUSTER_ALIAS}".internal"

CONSUME_TIMEOUT_MS=5000
EXCLUDE_STR1='"topic":"heartbeats"'
EXCLUDE_STR2='"group":'
INCLUDE_STR='"offset":'

IFS=$'\n'

output=( $(/engn/dc2/confluent-7.3.2/bin/kafka-console-consumer  --bootstrap-server sr-ksql01:9092 --from-beginning --property print.key=true --property key.separator ="-" --topic mm2-offsets.source.internal  --timeout-ms=${CONSUME_TIMEOUT_MS} | awk -F '[{,}]' '{print "{"$5"},{"$4"}]-{"$7"}"}') )

echo "processing ${#output[@]} messages from connect-offsets topic"

for (( i=0; i<${#output[@]}; i++ ))
 do
   if [[ "${output[$i]}" != *"$EXCLUDE_STR1"* && "${output[$i]}" != *"$EXCLUDE_STR2"* && "${output[$i]}" == *"$INCLUDE_STR"* ]]; then
   echo ${output[$i]}
   echo ${output[$i]} | /engn/dc2/confluent-7.3.2/bin/kafka-console-producer --broker-list ${BROKER_LIST} --topic ${REPLICATOR_OFFSET_TOPIC} --property parse.key=true --property key.separator="-"
   fi
 done
```

</aside>

---

# +Error.

- mm2 가 실행이 안되는 경우: mm2-offsets.source.internal 의 delete.policy = compact 로 생성되었는지 확인 필요

```bash
org.apache.kafka.common.config.ConfigException: Topic 'mm2-offsets.source.internal' supplied via the 'offset.storage.topic' property is required to have 'cleanup.policy=compact' to guarantee consistency and durability of source connector offsets, but found the topic currently has 'cleanup.policy=delete'. Continuing would likely result in eventually losing source connector offsets and problems restarting this Connect cluster in the future. Change the 'offset.storage.topic' property in the Connect worker configurations to use a topic with 'cleanup.policy=compact'.
        at org.apache.kafka.connect.util.TopicAdmin.verifyTopicCleanupPolicyOnlyCompact(TopicAdmin.java:521)
        at org.apache.kafka.connect.storage.KafkaOffsetBackingStore.lambda$initialize$2(KafkaOffsetBackingStore.java:247)
        at org.apache.kafka.connect.util.KafkaBasedLog.start(KafkaBasedLog.java:286)
        at org.apache.kafka.connect.storage.KafkaOffsetBackingStore.start(KafkaOffsetBackingStore.java:257)
        at org.apache.kafka.connect.runtime.Worker.start(Worker.java:229)
        at org.apache.kafka.connect.runtime.AbstractHerder.startServices(AbstractHerder.java:147)
        at org.apache.kafka.connect.runtime.distributed.DistributedHerder.run(DistributedHerder.java:342)
        at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:539)
        at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
        at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
        at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
        at java.base/java.lang.Thread.run(Thread.java:833)
```