# Docker HDFS HA

> hadoop version : 3.2.0
>
> ubuntu version : 18.04
>
> Namenode : No.2
>
> Datanode : No.5
>
> Journalnode : No.3
>
> 도커 환경에서  하둡 클러스터를 완전 분산 모드로 설정하는 방법을 기술한다.
>
> 또한 하둡은 HA로 구성하여 가용성을 더 높인다.
>
> ***"2021-11-23"***

### 목차

**[1. Docker 컨테이너 구축](#Docker-컨테이너-구축)**

**[2. HDFS 설정 파일 수정](#HDFS-설정-파일-수정)**

**[3. Hadoop 구동](#Hadoop-구동)**

**[4. 구동 오류 솔루션](#구동-오류-솔루션)**



## Docker 컨테이너 구축

- [**Docker 설치**](https://github.com/YounHS/Big_Data_Analysis_Platform/blob/main/1.%20Cloud/How%20to%20install%20Docker.md)

- **Docker group에 사용자 추가**

  ```bash
  $ sudo groupadd docker
  $ sudo usermod -aG docker $USER

  # 위 command 실행 후, reboot 권장
  ```


- **Docker 컨테이너 생성 및 실행**

  ```bash
  $ docker run --privileged -it -h master --name master -p 50070:50070 captainteemo/hd320hi312zk370:2.0 /sbin/init &
  $ docker run --privileged -it -h master2 --name master2 --link master:master captainteemo/hd320hi312zk370:2.0 /sbin/init &
  $ docker run --privileged -it -h worker1 --name worker1 --link master:master captainteemo/hd320hi312zk370:2.0 /sbin/init &
  $ docker run --privileged -it -h worker2 --name worker2 --link master:master captainteemo/hd320hi312zk370:2.0 /sbin/init &
  $ docker run --privileged -it -h worker3 --name worker3 --link master:master captainteemo/hd320hi312zk370:2.0 /sbin/init &

  # 이후 컨테이너 접속 시 하단의 command 사용
  $ docker exec -it master /bin/bash
  $ docker exec -it master2 /bin/bash
  $ docker exec -it worker1 /bin/bash
  $ docker exec -it worker2 /bin/bash
  $ docker exec -it worker3 /bin/bash
  ```

- **Docker container IP 확인**

  ```bash
  $ docker ps -a

  # 상기 command 결과로 확인한 Container ID를 통해 각 컨테이너 IP 숙지
  $ docker inspect {CONTAINER_ID}
  ```


<br>

## HDFS 설정 파일 수정

- **호스트 파일 수정**

  ```bash
  # 초기화되었을 시, 하단 구문 추가
  # 모든 컨테이너에서 수행
  $ vim /etc/hosts
  ...
  {CONTAINER_IP}	{CONTAINER_HOSTNAME}

  # ex)
  ...
  172.17.0.2      master
  172.17.0.3      master2

  172.17.0.4      worker1
  172.17.0.5      worker2
  172.17.0.6      worker3
  ```

- **workers 파일 수정**

  ```bash
  # 초기화되었을 시, worker 노드로 지정할 hostname 구문 추가
  # 모든 컨테이너에서 수행
  $ vim $HADOOP_HOME/etc/hadoop/workers
  master
  master2
  worker1
  worker2
  worker3
  ```

- **Datanode 디렉터리 경로 변경**

  ```bash
  # Datanode의 디렉터리 경로 및 이름 재설정
  # Datanode의 디렉터리경로/디렉터리명 동일 시, Datanode를 하나밖에 잡지 못함
  # 모든 컨테이너에서 수행
  $ vim $HADOOP_HOME/etc/hadoop/hdfs-site.xml
  ~
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>/hadoop_home/datanode_dir_master</value>
    <final>true</final>
  </property>
  ~

  # 상기에서 <property> 내의 <value>에서 "datanode_dir_master"에서 'master'를 각 컨테이너의 호스트명으로 변경
  ```

- **Datanode 디렉터리명 변경**

  ```bash
  # 모든 컨테이너에서 수행

  # 상기에서 작업에서 변경한 디렉터리명을 적용
  $ mv /hadoop_home/datanode_dir_master /hadoop_home/datanode_dir_{hostname}

  # 필요없는 디렉터리 삭제
  $ rm -rf /hadoop_home/datanode_dir
  ```

- **Namenode 초기화 전작업**

  ```bash
  # Namenode 초기화를 위해 journalnode의 디렉터리를 비워야함
  # Zookeeper가 설치된 서버에만 적용
  $ rm -rf /hadoop_home/journalnode_dir

  # Zookeeper로 사용하지 않을 서버에는 Zookeeper 제거
  # docker 업로드 당시, master, master2, worker1 의 호스트명을 가진 서버를 Zookeeper 서버로 설정했기 때문에 이에 맞춰 구축하는 것을 권장함
  $ rm -rf /zookeeper_home
  $ rm -rf /hadoop_home/journalnode_dir

  # 혹시 모를 충돌 대비 /tmp 디렉터리 내부 emptying
  $ rm -rf /tmp/*
  ```


<br>

## Hadoop 구동

- **방화벽 설정 해제**

  ```bash
  # Active Namenode 포맷을 위해 필요
  # 모든 컨테이너에서 수행
  $ systemctl stop firewalld
  ```

- **Zookeeper myid 수정**

  ```bash
  # master2, worker1만 수정
  # 각각 2, 3으로 변경하여 저장
  $ vim /zookeeper_home/apache-zookeeper-3.7.0-bin/conf/myid

  # 불필요한 파일 삭제
  $ rm -rf /zookeeper_home/apache-zookeeper-3.7.0-bin/conf/version-2
  $ rm -rf /zookeeper_home/apache-zookeeper-3.7.0-bin/conf/zookeeper_server.pid
  ```

- **Zookeeper 서버 구동**

  ```bash
  # Zookeeper가 설치된 서버에서 수행
  $ /zookeeper_home/apache-zookeeper-3.7.0-bin/bin/zkServer.sh start
  ```

- **Zookeeper 포맷 및 구동**

  ```bash
  # namenode 서버 중 한 곳에서
  $ hdfs zkfc -formatZK

  # 모든 namenode 서버에서
  $ hdfs --daemon start zkfc
  ```


- **JournalNode 구동**

  ```bash
  # Zookeeper가 설치된 서버에서 수행
  $ hdfs --daemon start journalnode
  ```

- **Namenode 초기화**

  ```bash
  # ActiveNode로 사용할 Namenode 서버 초기화
  # 디렉터리를 초기화할거냐고 물어보는데 대문자를 사용하는게 중요함
  $ hdfs namenode -format

  # 초기화 후, ActiveNode 구동
  $ hdfs --daemon start namenode

  # StandbyNode로 사용할 Namenode 서버를 ActiveNode에 bootstrap
  $ hdfs namenode -bootstrapStandby

  # StandbyNode 구동
  $ hdfs --daemon start namenode
  ```


- **전체 HDFS 구동**

  ```bash
  $ /hadoop_home/hadoop-3.2.0/sbin/start-all.sh
  ```

<br>

## 구동 오류 솔루션

- **DataNode 구동 오류**

  ```bash
  # Datanode 디렉터리 내, current 디렉터리 내 VERSION 파일 수정 필요
  # ClusterID 값이 namenode와 일치하지 않아 발생하는 문제

  # 하단의 명령어로 마지막 부분에 출력된 오류 확인
  $ cat $HADOOP_HOME/logs/hadoop-root-datanode-{hostname}.log
  ...
  java.io.IOException: Incompatible clusterIDs in /hadoop_home/datanode_dir_master: namenode clusterID = CID-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX; datanode clusterID = CID-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  ...

  # 상기에서 "namenode clusterID" 값을 복사한 후 하단의 명령어 실행
  # 편집기가 뜨면 하단의 clusterID 값을 복사한 "namenode clusterID" 값으로 대체
  $ vi /hadoop_home/datanode_dir_{hostname}/current/VERSION
  ...
  clusterID=CID-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
  ...

  # 하둡 재구동
  $ /hadoop_home/hadoop-3.2.0/sbin/start-all.sh
  ```

