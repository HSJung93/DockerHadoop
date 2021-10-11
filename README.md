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
  ![name_node](./img/name_node.png)

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
- `docker container ls`로 확인해보면 모든 컨테이너가 종료되었다.
