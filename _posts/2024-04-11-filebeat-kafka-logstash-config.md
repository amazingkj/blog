---
title: "[Kafka] filebeat, kafka, logstash 설정값과 설명"
excerpt: "filebeat, kafka, logstash 설정값과 설명"

categories:
  - Categories2
tags:
  - [filebeat, kafka, logstash]

permalink: /etc/filebeat-kafka-logstash-config/

toc: true
toc_sticky: true

date: 2024-04-11
last_modified_at: 2024-04-11
---

### filebeat.yml 파일 기본 설정 값

```yaml
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]

  # message topic selection + partitioning
  topic: '%{[fields.log_topic]}'
  partition.round_robin:
    reachable_only: true

  required_acks: -1
  compression: gzip
  max_retries: 3
  max_message_bytes: 1000000

```

- `partition.round_robin` : `round_robin` 설정.
    - `random`, `round_robin`, `hash` 중 파티셔닝 전략 선택 가능, 기본값은 `hash`
    - 파티션 1개 컨슈머 1개로 운영하기 때문에 큰 차이가 없으나, 클러스터로 운용할 경우 파티션에 데이터가 균등하게 배분되는 `round_robin` 이 가장 안정적일 것이기 때문에 채택
- `reachable_only` : `true`, 접근 가능한 브로커에 접근
    - `false`, 데이터가 균등하게 배분되다가 한쪽 브로커가 사용 불가할 경우 전송 불가
    - *cf ) All partitioners will try to publish events to all partitions by default. If a partition’s leader becomes unreachable for the beat, the output might block. All partitioners support setting `reachable_only`to overwrite this behavior. If `reachable_only`is set to `true`, events will be published to available partitions only.*
- `compression` : compression codec `none`, `snappy`, `lz4`, `gzip`중 선택. 프로듀서에서 kafka로 데이터가 압축해서 들어오는데 그 형식을 설정함
    - 기본값 **`gzip`**
- `max_message_bytes` : 메시지를 보내는 최대 바이트
    - 기본값 1000000 (bytes)
- `required_acks` : 기본값은 1, **idempotence(멱등법칙)를 위하여 -1로 설정.**
    - 0인 경우, 메세지가 카프카에 잘 전달됐는지 확인 X : 성능은 빠르나 데이터 유실 가능성 有
    - 1인 경우, leader Partition만 확인 : 메시지 소실은 없지만 중복해서 데이터를 받는 일이 생김 (적어도 한번 전송)
    - 1인 경우, replica까지 잘 전달됐는지 확인. PID, SEQ를 헤더에 묶어서 보내고 중복 데이터일 경우 seq 확인해서 최근 데이터로 업데이트(중복 방지), 성능은 최대 20% 감소, 카프카 버전 3.0부터는 기본 설정
- `max_retries` : 전송에 실패했을때 retry 횟수
    - 기본값 3



### logstash.conf 파일 기본 설정값

```yaml
input {
    kafka {
        bootstrap_servers => "192.168.0.103:9092"
        topics => "test-log-1"
        consumer_threads => 1
    }
}

```

- `fetch_max_bytes` : 한 번에 가져올 수 있는 최대 데이터
    - 기본값 52428800 (50MB)
- `fetch_max_wait_ms` : fetch_min_bytes 이상의 메시지가 쌓일 때까지 최대 대기 시간.
    - 기본값 500 (ms)
- `fetch_min_bytes` : fetcher가 record들을 읽어 들이는 최소 byte수
    - 기본값 없음, `fetch_max_wait_ms`와 `fetch_min_bytes`는 둘 중 하나만 만족하면 데이터를 읽기 시작함.
- `heartbeat_interval_ms` : Heart Beat Thread가 Heart Beat을 보내는 간격.
    - kafka Group Coordinator 에서 컨슈머에게 신호를 보내 응답 신호인 HeartBeat를 통해 컨슈머가 동작하고 있음을 체크
    - 기본값 3000 (ms)
- `session_timeout_ms` : 브로커가 Consumer로 Heart Beat을 기다리는 최대 시간
    - 기본값 10000 (ms), `heartbeat_interval_ms` 보다는 값이 커야 함
- `max_poll_interval_ms` : 이전 poll() 호출 후 다음 호출 poll()까지 브로커가 기다리는 시간
    - 기본값 500000 (ms)
- `enable_auto_commit` : `true`인 경우 읽어온 메시지를 브로커에 바로 commit 적용하지 않고, auto_commit_interval_ms 에 정해진 주기마다 Consumer가 자동으로 Commit을 수행, Commit 하는 정보는 데이터를 어디까지 읽었는가에 대한 내용으로 kafka offset에 저장. (컨슈머 그룹별로 저장, 그룹마다 offset은 하나)
- `auto_commit_interval_ms` : 기본값 5000 (ms)