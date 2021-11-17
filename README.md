# 도커를 이용한 하둡 클러스터 만들기

## 도커 컴포즈를 이용한 복수의 컨테이너들 설치

- `git clone https://github.com/big-data-europe/docker-hadoop.git`으로 프로젝트 파일을 가져온다.
- `cd docker-hadoop`으로 이동한 후 `docker-compose up`으로 도커에서 복수의 컨테이너를 실행시킨다.
  - 도커 컴포즈는 도커에서 복수의 컨테이너를 실행시키는 툴이다. `docker-compose.yml`이라는 YAML 파일을 사용하여 어플리케이션의 서비스를 구성한다. `docker-compose up` 명령어를 실행하여 컴포즈를 실행시키고 전체 앱을 실행시킨다.
- `docker container ls`로 컨테이너들을 확인한다.

```
CONTAINER ID   IMAGE                                                    COMMAND                  CREATED          STATUS                    PORTS
                NAMES
b9fa2d4c7eb4   bde2020/hadoop-historyserver:2.0.0-hadoop3.2.1-java8     "/entrypoint.sh /run…"   13 minutes ago   Up 12 minutes (healthy)   8188/tcp
                historyserver
a9fbc41e1f45   bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8   "/entrypoint.sh /run…"   13 minutes ago   Up 12 minutes (healthy)   8088/tcp
                resourcemanager
9aee63cbdeb4   bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   13 minutes ago   Up 12 minutes (healthy)   0.0.0.0:9000->9000/tcp, 0.0.0.0:9870->9870/tcp   namenode
ea79103308f0   bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8       "/entrypoint.sh /run…"   13 minutes ago   Up 12 minutes (healthy)   8042/tcp
                nodemanager
0eee7d8613ca   bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   13 minutes ago   Up 12 minutes (healthy)   9864/tcp
                datanode
```

- namenode(0.0.0.0:9000->9000, 0.0.0.0:9870->9870), datanode(9864), nodemanager(8042), resourcemanager(8088), historyserver(8188)를 확인할 수 있다.

## 네임노드 컨테이너 접속 및 디렉토리 생성 명령

- `docker exec -it namenode /bin/bash`로 네임노드에 들어간다.
  - `docker exec`는 container에 특정 명령을 실행시키라는 명령이고, 이때 실행시킬 명령이 /bin/bash이다.
  - `-it`는 STDIN 표준 입출력을 열고 가상 tty (pseudo-TTY) 를 통해 접속하겠다는 의미이다.
  - 네임노드 컨테이너에 접속 성공: `root@9aee63cbdeb4:/#`
- `hdfs dfs -ls /` 명령어를 사용하여 디렉토리를 확인한다.
  - 하둡 파일 시스템에서는 `hdfs dfs [일반적인 옵션] [커맨드 옵션]`으로 명령어를 내린다.
    - 예시) 로컬 파일을 hdfs에 저장한다: `-appendToFile`, 해당 파일의 그룹 권한을 변경한다: `-chmod`, hdfs의 파일을 로컬 디렉토리에 다운로드한다: `-copyToLocal`, hdfs 내부에서 파일을 복사하고 붙여넣기 한다: `-cp` 등
- `hdfs dfs -mkdir -p /user/root`로 디렉토리를 생성한다.
- `exit`로 접속을 종료한다.

## 이더넷(WSL) 접속

- `ipconfig`로 이더넷 어댑터(WSL)의 ipv4를 확인한다.
- 웹 브라우저로 해당 ip의 :9870 포트로 접속한다.
- 다음과 같은 화면이 나오면 성공적으로 접속한 것이다.
  ![name_node](img/name_node.PNG)

## jar 파일을 다운 및 네임노드 컨테이너에 복사하기

- `wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-mapreduce-examples/2.7.1/hadoop-mapreduce-examples-2.7.1-sources.jar`로 파일 다운로드(리눅스)
- `docker cp hadoop-mapreduce-examples-2.7.1-sources.jar namenode:/tmp/`로 로컬에서 컨테이너로 파일을 복제한다.
- 테스트에 사용할 `input1.txt`를 만든 뒤 `docker cp input1.txt namenode:/tmp/`로 복제한다.

## hdfs에 파일 업로드

- `docker exec -it namenode /bin/bash`로 다시 네임노드 컨테이너에 명령어를 수행한다.
- `cd /tmp/`로 이동 후, `cat input1.txt`로 확인하면 해당 파일이 복제된 것을 확인할 수 있다.
- input1.txt 파일을 hdfs로 업로드 하기 위해 `hdfs dfs -mkdir /user/root/input` 명령어로 hdfs에 디렉토리를 만든다.
- `hdfs dfs -put input1.txt /user/root/input/`로 input1.txt 파일을 hdfs에 업로드 한다.
- 역시 `hdfs dfs -cat /user/root/input/input1.txt`로 확인할 수 있다.

## 맵리듀스 작업 실행하기

- hadoop jar 커맨드로 맵리듀스 jar 파일을 실행하고 클래스 파일을 불러온다. 만들어 놓은 input 디렉토리를 사용하고 output 디렉토리를 생성한다. : `hadoop jar hadoop-mapreduce-examples-2.7.1-sources.jar org.apache.hadoop.examples.WordCount input output`
- 다음과 같은 명령어가 출력되면 성공적으로 맵리듀스 작업을 실행한 것이다.

```
2021-10-11 12:01:05,955 INFO client.RMProxy: Connecting to ResourceManager at resourcemanager/172.18.0.3:8032
2021-10-11 12:01:06,091 INFO client.AHSProxy: Connecting to Application History server at historyserver/172.18.0.6:10200
2021-10-11 12:01:06,242 INFO mapreduce.JobResourceUploader: Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/root/.staging/job_1633947212148_0001
2021-10-11 12:01:06,321 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2021-10-11 12:01:06,412 INFO input.FileInputFormat: Total input files to process : 1
2021-10-11 12:01:06,434 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2021-10-11 12:01:06,451 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2021-10-11 12:01:06,458 INFO mapreduce.JobSubmitter: number of splits:1
2021-10-11 12:01:06,543 INFO sasl.SaslDataTransferClient: SASL encryption trust check: localHostTrusted = false, remoteHostTrusted = false
2021-10-11 12:01:06,556 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1633947212148_0001
2021-10-11 12:01:06,556 INFO mapreduce.JobSubmitter: Executing with tokens: []
2021-10-11 12:01:06,698 INFO conf.Configuration: resource-types.xml not found
2021-10-11 12:01:06,698 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
2021-10-11 12:01:07,094 INFO impl.YarnClientImpl: Submitted application application_1633947212148_0001
2021-10-11 12:01:07,127 INFO mapreduce.Job: The url to track the job: http://resourcemanager:8088/proxy/application_1633947212148_0001/
2021-10-11 12:01:07,127 INFO mapreduce.Job: Running job: job_1633947212148_0001
2021-10-11 12:01:13,200 INFO mapreduce.Job: Job job_1633947212148_0001 running in uber mode : false
2021-10-11 12:01:13,201 INFO mapreduce.Job:  map 0% reduce 0%
2021-10-11 12:01:18,246 INFO mapreduce.Job:  map 100% reduce 0%
2021-10-11 12:01:22,279 INFO mapreduce.Job:  map 100% reduce 100%
2021-10-11 12:01:22,292 INFO mapreduce.Job: Job job_1633947212148_0001 completed successfully
2021-10-11 12:01:22,361 INFO mapreduce.Job: Counters: 54
        File System Counters
                FILE: Number of bytes read=100
                FILE: Number of bytes written=458745
                FILE: Number of read operations=0
                FILE: Number of large read operations=0
                FILE: Number of write operations=0
                HDFS: Number of bytes read=190
                HDFS: Number of bytes written=76
                HDFS: Number of read operations=8
                HDFS: Number of large read operations=0
                HDFS: Number of write operations=2
                HDFS: Number of bytes read erasure-coded=0
        Job Counters
                Launched map tasks=1
                Launched reduce tasks=1
                Rack-local map tasks=1
                Total time spent by all maps in occupied slots (ms)=6436
                Total time spent by all reduces in occupied slots (ms)=13056
                Total time spent by all map tasks (ms)=1609
                Total time spent by all reduce tasks (ms)=1632
                Total vcore-milliseconds taken by all map tasks=1609
                Total vcore-milliseconds taken by all reduce tasks=1632
                Total megabyte-milliseconds taken by all map tasks=6590464
                Total megabyte-milliseconds taken by all reduce tasks=13369344
        Map-Reduce Framework
                Map input records=3
                Map output records=13
                Map output bytes=128
                Map output materialized bytes=92
                Input split bytes=112
                Combine input records=13
                Combine output records=10
                Reduce input groups=10
                Reduce shuffle bytes=92
                Reduce input records=10
                Reduce output records=10
                Spilled Records=20
                Shuffled Maps =1
                Failed Shuffles=0
                Merged Map outputs=1
                GC time elapsed (ms)=71
                CPU time spent (ms)=720
                Physical memory (bytes) snapshot=548495360
                Virtual memory (bytes) snapshot=13576130560
                Total committed heap usage (bytes)=489684992
                Peak Map Physical memory (bytes)=318685184
                Peak Map Virtual memory (bytes)=5113307136
                Peak Reduce Physical memory (bytes)=229810176
                Peak Reduce Virtual memory (bytes)=8462823424
        Shuffle Errors
                BAD_ID=0
                CONNECTION=0
                IO_ERROR=0
                WRONG_LENGTH=0
                WRONG_MAP=0
                WRONG_REDUCE=0
        File Input Format Counters
                Bytes Read=78
        File Output Format Counters
                Bytes Written=76
```

- exit로 네임노드 컨테이너에서 접속을 종료한다.

## 도커 컴포즈로 컨테이너 종료

- `docker-compose down`으로 모든 컨테이너를 종료할 수 있다.

```
[+] Running 6/6
 - Container historyserver        Removed                                                                                                    11.6s
 - Container namenode             Removed                                                                                                    12.2s
 - Container datanode             Removed                                                                                                    12.1s
 - Container resourcemanager      Removed                                                                                                    11.9s
 - Container nodemanager          Removed                                                                                                    11.6s
 - Network docker-hadoop_default  Removed                                                                                                     0.6s
```

- `docker container ls`로 확인해보면 모든 컨테이너가 종료되었다.

# 도커 바인드 마운트

## 개요

- 도커 컨테이너에 쓰여진 데이터는 기본적으로 컨테이너가 삭제될 때 사라지게 된다. 따라서 컨테이너의 생명 주기와 관련없이 데이터를 영속적으로 저장하고, 여러 개의 컨테이너가 공유할 저장 공간이 필요하게 된다.
- 도커는 이를 위해 1. 볼륨과 2. 바인드 마운트를 제공한다. 볼륨과 바인드 마운트의 가장 큰 차이점은 도커가 해당 마운트 포인트를 관리해주는지 안해주는지의 차이이다. 볼륨을 사용할 때는 개발자가 직접 볼륨을 생성하거나 삭제해야하지만, 해당 볼륨을 도커 상에서 관리가 되는 이점이 있다.

## index.html 파일 바인드 마운트 하기

- `docker pull nginx`로 nginx 이미지를 다운 받는다.
- `cd bind-mount`로 이동한다.
- 컨테이너로 마운트 하기 위하여 호스트에 `mkdir -p /tmp/nginx/html` 명령어로 디렉토리를 만든다.
- `docker run -t -d -P -v /tmp/nginx/html:/usr/share/nginx/html --name nginxcontainer nginx:latest`
  - `-d`는 detachable 모드에서 실행을 의미한다. 컨테이너에가 백그라운드에서 실행되며 실행 결과로 컨테이너 ID만을 출력한다. 터미널에서 빠져나와도 해당 컨테이너가 종료되지 않는다.
  - `-it`는 컨테이너를 종료하지 않은 채로 터미널의 입력을 계속해서 컨테이너로 전달하기 위해서 사용한다.
  - `--name` 옵션으로 컨테이너에 이름을 부여한다.
  - `-p` 옵션으로 호스트와 컨테이너 간의 포트 배포/바인드를 한다.
  - `-v` 옵션은 호스트와 컨테이너 간의 볼륨을 설정한다. 호스트 컴퓨터 파일 시스템의 특정 결로를 컨테이너 파일 시스템의 특정 경로로 마운트 한다.
- `docker container ls`로 확인한다.

```
CONTAINER ID   IMAGE          COMMAND                  CREATED          STATUS          PORTS                   NAMES
4fe26cbad64c   nginx:latest   "/docker-entrypoint.…"   16 seconds ago   Up 15 seconds   0.0.0.0:49153->80/tcp   nginxcontainer
```

- `ipconfig`로 ip주소를 확인하고 앞서 확인한 포트 넘버를 더해 웹으로 접속해보면 아직은 403 Forbidden 화면이 나온다. 아직 컨테이너에 html 파일이 없으므로 당연한 결과이다.
- index.html 파일을 `/tmp/nginx/html`에 만들고 수정하면 해당 파일이 연동된 것을 확인할 수 있다.
- `docker inspect 4fe26cbad64c`로 "Mounts"의 이미지를 확인할 수 있다.

# 도커 하이브

- `mkdir docker-hive`로 생성 `cd docker-hive`로 이동 후 `git clone https://github.com/big-data-europe/docker-hive.git`로 해당 파일 다운로드한다.
- `ls`로 `docker-compose.yml` 파일 등을 확인하면 여러 서비스(namenode, datanode, hive-server 등)에 대응되는 컨테이너들을 확인할 수 있다.
- `docker-compose up -d`로 이미지들을 다운로드하고 컨테이너들을 다운로드 한다. `docker container ls`로 모두 업 되고 러닝되는 것을 확인한다.

```
CONTAINER ID   IMAGE                                                    COMMAND                  CREATED              STATUS                        PORTS
                        NAMES
2a90c7220b94   bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8       "/entrypoint.sh /run…"   About a minute ago   Up About a minute (healthy)   8042/tcp
                        nodemanager
470340b10ef9   bde2020/hadoop-historyserver:2.0.0-hadoop3.2.1-java8     "/entrypoint.sh /run…"   About a minute ago   Up About a minute (healthy)   8188/tcp
                        historyserver
43c519dca253   bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8   "/entrypoint.sh /run…"   About a minute ago   Up 41 seconds (healthy)       8088/tcp
                        resourcemanager
4d4c2b768c85   bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   About a minute ago   Up About a minute (healthy)   0.0.0.0:9000->9000/tcp, 0.0.0.0:9870->9870/tcp   namenode
addd9382864c   bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8          "/entrypoint.sh /run…"   About a minute ago   Up About a minute (healthy)   9864/tcp
                        datanode
```

- `docker-compose exec hive-server bash`로 하이브 서버에 컨테이너에 접속한다.
- `/opt/hive/beeline -u jdbc:hive2://localhost:10000` 명령어로 db를 조작하기 시작한다.
- `show databases ;`로 보면 db가 없으므로 `create database training ;`로 만들고 `use training ;`로 db를 만든 뒤에 `create table fruits (name string, price int) ;` 로 테이블을 만든다. `show tables;`와 `describe fruits ;`로 확인한다.
- `describe formatted fruits ;` 명령어로 데이터의 위치를 파악할 수 있다:

```
Location: hdfs://namenode:8020/user/hive/warehouse/training.db/fruits
```

- `!quit` 명령어로 hive를 종료하고 `exit`로 컨테이너로 부터 나온다.
- `docker compose down`으로 컨테이너를 종료한다.

# 도커 카프카

- 각기 다른 포트로 3개의 주키퍼와 카프카 서비스를 동작시키는 docker-compose.yml 파일을 작성한다.
- 도커허브의 confluentinc/cp-kafka 이미지를 사용한다.
- `docker-compose up -d` 명령어로 detachable 모드로 실행한다.
- `docker container ls`로 확인할 수 있다.
- `telnet localhost 12181`, `telnet localhost 22181`, `telnet localhost 32181`, `telnet localhost 19092`
- `docker-compose down`으로 종료한다.

# 이론

## 하둡

### 하둡 정의

- 컴퓨터 여러 대를 클러스터화하고, 큰 크기의 데이터를 클러스터에서 병렬로 동시에 처리하여 처리 속도를 높이는 것을 목적으로 하는 분산처리를 위한 오픈소스 프레임워크

### 하둡의 구성요소

- Common: 다른 모듈을 지원하기 위한 공통 컴포넌트 모듈
- HDFS: 여러개의 서버를 하나의 서버처럼 묶어서 데이터를 분산 저장하는 모듈
- YARN: 클러스터 자원 관리 및 스케줄링 담당
- Mapreduce: 분산 저장된 데이터를 분산 처리하는 모듈
- Ozone: 오브젝트 저장소

### 하둡의 장단점

- 장점:
  - 시스템을 중단하지 않고 장비의 추가가 용이하다
  - 일부 장비에 장애가 발생하더라도 전체 시스템 사용성에 영향이 적다
  - 저렴한 구축 비용
- 단점:
  - HDFS에 저장된 데이터를 변경 불가
  - 실시간 데이터 분석과 같이 신속한 처리 작업에 부적합

### 하둡 버전 별 특징

- v1:
  - HDFS로 분산 저장: 네임노드, 데이터 노드
  - MapReduce로 병렬 처리: 잡 트래커, 테스트 트래커
- v2:
  - YARN 도입하여 잡 트래커의 병목 현상 제거. 잡 트래커의 기능을 자원 관리, 라이프 사이클 관리, 작업의 처리로 분리.
  - 자원 관리와: 리소스 매니저와 노드 매니저
  - 라이프 사이클 관리: 애플리케이션 마스터
  - 작업의 처리: 컨테이너
- v3:
  - 패리티 블록을 이용한 이레이져 코딩 도입하여 HDFS 데이터 저장의 효율성 증가
  - 2개 이상의 네임 노드 지원
  - 자바 8 지원.

### HDFS

- 블록 단위 저장
- 블록 복제를 이용한 장애 복구
- 읽기 중심, 파일의 수정을 지원하지 않는다.
- 데이터 지역성 이용.
- 네임노드와 데이터노드로 구성

### 네임노드

- 메타데이터 관리

  - 파일이름, 파일크기, 파일생성시간, 접근 권한, 위치 등
  - dfs.name.dir에 보관
  - 수정사항을 네임노드의 메모리에 바로 적용, Edist 파일에 주기적 저장
  - Fsimage 파일: 네임스페이스와 블록 정보
  - Edits 파일: 파일의 생성, 삭제에 대한 트랜잭션 로그

- 데이터 노드 관리

  - 3초 주기의 하트비트, 6시간 주기의 블록 리포트를 이용하여 관리
  - 하트비트가 도착하지 않으면 네임노드는 데이터노드가 동작 하지 않는 것으로 간주하고 더이상 IO가 발생하지 않도록 조치한다.
  - 블록 리포트로 저장된 파일이 어떤 블록이고 어디 저장되어 있는지에 대한 최신 정보 유지.

- 구동 과정
  - Fsimage를 읽어 메모리에 적재
  - Edits 파일로 변경 내역 반영
  - 현재 메모리 상태를 스냅샷으로 Fsimage 생성
  - 데이터 노드로부터 블록 리포트 수신하여 매핑
  - 서비스 시작
    - 네임노드에 파일이 보관된 블록 위치 요청하고 반환
    - 각 데이터 노드에 파일 블록 요청
    - 블록 깨져 있으면 통지하고 다른 블록 확인

### 데이터노드

- 활성/비활성 상태
  - 하트비트를 주기적으로 받는 상태면 활성
- 운영 상태
  - normal, decommissioned, in_maintenance

### 블록지역성

- 맵리듀스를 처리할 때 현재 노드에 저장되어 있는 블록을 이용하는 것.

### 블록 캐싱

- 데이터 노드에 저장된 데이터 중 자주 읽는 블록은 블록 캐시라는 데이터노드의 메모리에 명시적으로 캐싱하는 것.

### 세컨더리 네임 노드

- Fsimage와 Edits 파일을 주기적으로 머지하여 최신 블록의 상태로 파일을 생성한다.

### HDFS 페더레이션

- 디렉토리(네임스페이스) 단위로 네임노드를 등록하여 사용하는 것.

### HDFS 고가용성

- HDFS 고가용성은 이중화된 두대의 서버인 액티브 네임노드와 스탠바이 네임노드로 지원한다.
- 하트비트를 모두 받아서 동일한 메타데이터를 유지하고, 공유 스토리지를 이용하여 에디트 파일을 공유한다.
- 스탠바이 네임노드는 액티브 네임노드와 동일한 메타데이터 정보를 유지하다가, 액티브 네임노드에 문제가 발생하면 스탠바이 네임노드가 액티브 네임노드로 동작
- 주키퍼를 이용하여 장애 발생시 자동으로 변경될 수 있도록 한다.
- 스탠바이 네임노드는 세컨더리 네임노드의 역할을 동일하게 수행하므로, HDFS 고가용성 모드로 설정하였을 때는 세컨더리 네임노드를 실행하지 않아도 되고 실행하면 오류가 발생한다.

### HDFS 데이터 블록 관리

- 커럽트
  - 모든 복제 블록에 문제가 생겨서 복구하지 못하는 상태
  - 커럽트 상태의 파일들은 삭제하고 원본 파일을 다시 HDFS에 올려주어야 한다
  - `fsck`를 이용하여 상태를 체크한다
- 복제 개수 부족

### 맵리듀스

- 간단한 단위 작업을 반복하여 처리할 때 사용하는 프로그래밍 모델. 단위 작업을 처리하는 맵과, 모아서 집계하는 리듀스 단계로 구성.
- 실행 중 오류가 발생하면 설정된 횟수만큼 자동으로 반복된다. 반복 후에도 오류가 발생하면 작업을 종료한다.
- 맵 작업
  - 큰 데이터를 하나의 노드에서 처리하지 않고 분할하여 동시에 병렬 처리하여 작업 시간을 단축한다.
  - 맵 작업은 HDFS에 입력 데이터가 있는 노드에서 실행할 때, 클러스터의 네트워크 대역을 사용하지 않고 처리하기 때문에 가장 빠르게 동작한다.
  - 노드에서 작업을 처리할 수 없다면, 동일한 랙의 노드, 다른 랙의 노드 순서로 맵 작업이 실행가능한 노드를 찾는다.
- 리듀서 작업
  - 없는 경우: 파일을 읽어서 바로 쓰는 작업의 경우 매퍼만 있는 작업
  - 하나인 경우: 모든 데이터의 정렬 작업
  - 여러개인 경우: 리듀서 수만큼 파일 생성

### 맵리듀스의 처리단계

- 입력: text, csv, gzip 형태
  - `InputFormat`, `InputSplit`, `RecordReader` 추상클래스를 상속하여 구현
- 맵: 입력을 분할하여 키별 데이터 처리
  - `Mapper` 클래스를 상속하고 `map()` 메소드를 구현
- 컴바이너: 네트워크를 타고 넘어가는 데이터를 줄이는 로컬 리듀서
- 파티셔너: 맵의 출력 결과 키 값을 해쉬 처리하여 어떤 리듀서로 넘길지 결정.
  - `Partitioner` 클래스를 상속받아서
- 셔플: 각 리듀서로 데이터 이동
- 정렬: 리듀서로 전달된 데이터를 키 값 기준으로 정렬
  - 데이터를 정렬할 때 파티션의 기준이 되는 주키와 다른 값을 기준으로 정렬할 수 있다. 이를 세컨더리 소트라고 한다.
- 리듀서: 데이터를 처리하고 결과를 저장
- 출력: 리듀서의 결과를 정의된 형태로 저장

### 작은 파일 문제

- 네임노드는 파일의 메타데이터와 블록을 관리하는데 많은 메모리를 사용한다. 작은 사이즈의 파일이 여러개 존재하게 되면 이 파일들을 관리하는데 많은 메모리가 사용되고, 맵리듀스 작업 처리 중 많은 요청을 처리하게 되어 네임노드에 병목 현상이 발생하게 되어 작업속도가 느려지게 되니다. 이를 방지하기 위하여 작은 사이즈의 파일을 하나의 파일로 합쳐서 HDFS 블록사이즈 크기의 파일로 설정하는 것이 좋다.

### YARN

- 클러스터 리소스 관리 및 애플리케이션 라이프 사이클 관리를 위한 아키텍쳐이다.
- 잡트래커 한 대가 모든 클러스터의 모든 노드를 관리하면서, 메모리 자원 사용의 효율성 문제 발발. 맵리듀스 작업만 처리하고 SQL 기반 작업의 처리나 인메모리 기반의 작업 처리에 어려움이 있기 때문에 등장

### 자원 관리

- 리소스매니저
  - 전달 받은 정보를 이용하여 클러스터 전체의 자원 관리.
  - 애플리케이션 마스터에서 자원을 요청하면 비어 있는 자원을 사용할 수 있도록 처리
  - 스케줄러: `yarn-site.xml`에 설정
    - 피포 스케줄러: 먼저 들어온 작업이 종료될 때까지 다음 작업은 대기하는 테스트 목적으로만 사용하는 스케줄러
    - 페어 스케줄러: 동등하게 리소스 점유.
    - 커패시티 스케줄러: 트리 형태로 큐를 선언하고 각 큐 별로 이용할 수 있는 자원의 용량을 정하여 주면 그 용량에 맞게 자원을 할당한다.
- 노드매니저
  - 클러스터의 각 노드마다 실행. 현 노드의 자원 상태 관리 및 보고

### 라이프사이클 관리

- 애플리케이션 마스터
  - 클라이언트가 리소스 매니저에 애플리케이션을 제출하면 리소스 매니저가 비어있는 노드에서 애플리케이션 마스터를 실행시킨다.
  - 작업 실행을 위한 자원을 리소스 매니저에 요청하고 자원을 할당 받아서 각 노드에 컨테이너를 실행하고 작업 진행.
  - 컨테이너로부터 종료 알림을 받고 작업이 종료되면 리소스매니저에 알리고 자원 해제
- 컨테이너
  - 실제 작업이 실행되는 단위.
  - 작업 종료시 애플리케이션에 알림.
