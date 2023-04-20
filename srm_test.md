## SRM에서 test topic 제외
- CM 에서 black.list에 추가

## TEST TOPIC 생성

- test topic 생성 
    - name : cp-mig-test
    - replication.factor = 3, in.insync.replica = 2, partitions = 1

- 데이터 produce 200건


## SRM 으로 복제

- offset을  99으로 설정 후 복제 시작
```
#consume으로 데이터 형식 확인

$ ./kafka-console-producer.sh --bootstrap-server kafka-dc:9092 --topic mirrormaker2-cluster-offsets --property parse.key=true --property key.separator="|"
["aws->dc.MirrorSourceConnector",{"cluster":"m16cdc","partition":0,"topic":"cp-mig-test"}]|{"offset":99}
```
- CM 에서 black.list에서 제외

## 데이터 추가 생성
- 200건 추가 생성 

## target topic데이터 확인
- 300 건 데이터 확인

## SRM에서 test topic 제외 
- blacklist에 test topic추가

## replicator로 복제
- connect-offsets 토픽 생성 -> 
    - 이름 : replicator-connect-offsets
    - replication.factor = 3, in.insync.replica = 2, partitions = 1, cleanup.policy = compacted
- replicator 구성
    - 특정 토픽(cp-mig-test) 만 복제하도록  replicator 구성
    - connector-offset 토픽명 설정 : replicator-connect-offsets
    - 토픽 prefix 추가하도록 
- offset를 399으로 설정

```
/kafka-console-producer --bootstrap-server <target broker>:9092 --topic <connect-offsets 토픽명> --property parse.key=true --property key.separator="|"

>["replicator",{"topic":"cp-mig-test","partition":0}]|{"offset":399}
```

- replicator 시작 

## 데이터 추가 생성
- 200건 추가 생성 

## target topic데이터 확인
- 500 건 데이터 확인

## replicator 종료

## SRM으로 복구 
- offset을 599으로 변경
- black list 에서 제외

## 데이터 추가 생성
- 200건 추가 생성 

## target topic데이터 확인
- 700 건 데이터 확인

# 확인 사항
각 솔루션 별 connect-offsets 토픽내의 키, 밸류 데이터 형식
각 솔루션 별 black.list추가 방법
스크립트 작성 가능 여부


