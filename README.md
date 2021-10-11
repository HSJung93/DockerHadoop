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

## Supported Hadoop Versions

See repository branches for supported hadoop versions

## Quick Start

To deploy an example HDFS cluster, run:

```
  docker-compose up
```

Run example wordcount job:

```
  make wordcount
```

Or deploy in swarm:

```
docker stack deploy -c docker-compose-v3.yml hadoop
```

`docker-compose` creates a docker network that can be found by running `docker network list`, e.g. `dockerhadoop_default`.

Run `docker network inspect` on the network (e.g. `dockerhadoop_default`) to find the IP the hadoop interfaces are published on. Access these interfaces with the following URLs:

- Namenode: http://<dockerhadoop_IP_address>:9870/dfshealth.html#tab-overview
- History server: http://<dockerhadoop_IP_address>:8188/applicationhistory
- Datanode: http://<dockerhadoop_IP_address>:9864/
- Nodemanager: http://<dockerhadoop_IP_address>:8042/node
- Resource manager: http://<dockerhadoop_IP_address>:8088/

## Configure Environment Variables

The configuration parameters can be specified in the hadoop.env file or as environmental variables for specific services (e.g. namenode, datanode etc.):

```
  CORE_CONF_fs_defaultFS=hdfs://namenode:8020
```

CORE_CONF corresponds to core-site.xml. fs_defaultFS=hdfs://namenode:8020 will be transformed into:

```
  <property><name>fs.defaultFS</name><value>hdfs://namenode:8020</value></property>
```

To define dash inside a configuration parameter, use triple underscore, such as YARN*CONF_yarn_log\*\*\_aggregation*\*\*enable=true (yarn-site.xml):

```
  <property><name>yarn.log-aggregation-enable</name><value>true</value></property>
```

The available configurations are:

- /etc/hadoop/core-site.xml CORE_CONF
- /etc/hadoop/hdfs-site.xml HDFS_CONF
- /etc/hadoop/yarn-site.xml YARN_CONF
- /etc/hadoop/httpfs-site.xml HTTPFS_CONF
- /etc/hadoop/kms-site.xml KMS_CONF
- /etc/hadoop/mapred-site.xml MAPRED_CONF

If you need to extend some other configuration file, refer to base/entrypoint.sh bash script.
