# docker-kafka-example
Kafka with ZooKeeper Example for Docker Compose
<br><br>
- - -

## 준비사항  

프로젝트 클론 후, Dockerfile들의 COPY에 필요한 파일들을 준비한다.  

<https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz>  
위 페이지에서 받은 파일의 압축을 풀고, apache-zookeeper-3.8.0-bin 디렉터리의 이름을 zookeeper로 하여 함께 둔다.  
(./zookeeper/bin, ./zookeeper/conf, ...)

<https://www.apache.org/dyn/closer.cgi?path=/kafka/3.3.1/kafka_2.13-3.3.1.tgz>  
위 페이지에서 받은 파일의 압축을 풀고, kafka_2.13-3.3.1 디렉터리의 이름을 kafka로 하여 함께 둔다.  
(./kafka/bin, ./kafka/config, ...)  
<br>
![준비사항](./prepare.png)
<br><br>
- - -

## 서버 자동 실행 타입 (kafka-auto)  

kafka-auto 디렉터리로 이동한다.  
% `cd ./kafka-auto`  

docker-compose version은 v2를 사용 (예시에서는 v2.15.1 사용)  
% `docker compose up -d`  
로 빌드 및 실행한다.  
<br>
docker-compose 환경은 v2 기본값인 **buildkit 사용**이며,  
만약 **Docker Desktop** 환경에서 `ERROR [internal] load metadata for docker.io/library/...` 에러가 난다면,  
`~/.docker/config.json`에서 `"credStore": "desktop",` 부분을 `"credStore": "",`로 수정하고 Docker Desktop을 재시작  
<br><br>

`docker-compose.yml`의 하단부를 보면 알지만,  
컨테이너들 실행 후 약간의 대기 시간을 두고 토픽을 생성하고 produce/consume한다.  

그러므로 총 약 8초를 기다려준다.  
(ZooKeeper 실행, 상호 연결 및 leader/follower 선출, Kafka 실행 및 ZooKeeper 연결 등)  

터미널을 2개 열고, 하나는 컨슈머에 진입한다.  
% `docker attach kafka-auto-consumer`  

하나는 프로듀서로 진입한다.  
% `docker attach kafka-auto-producer`  
프로듀서에서 원하는 문자열들을 몇 줄 작성해본다.  
컨슈머에서 확인한다.  
<br><br>
- - -

## 서버 수동 실행 타입 (kafka-manual)  

kafka-manual 디렉터리로 이동한다.  
% `cd ./kafka-manual`  

docker-compose version은 v2를 사용 (예시에서는 v2.15.1 사용)  
% `docker compose up -d`  
로 빌드 및 실행한다.  
<br>
docker-compose 환경은 v2 기본값인 **buildkit 사용**이며,  
만약 **Docker Desktop** 환경에서 `ERROR [internal] load metadata for docker.io/library/...` 에러가 난다면,  
`~/.docker/config.json`에서 `"credStore": "desktop",` 부분을 `"credStore": "",`로 수정하고 Docker Desktop을 재시작  
<br><br>

% `docker attach kafka-manual-zookeeper-cont-01`  
내부로 진입하여  
&#35; `/zookeeper/bin/zkServer.sh start`  
그러면  
```
ZooKeeper JMX enabled by default
Using config: /zookeeper/bin/../conf/zoo.cfg  
Starting zookeeper ... FAILED TO START
```
`FAILED TO START`로 뜨지만,  
이 예시에서는 **멀티 클러스터 환경**으로 설정한 것이라 클러스터 수가 부족하여 그런 것 뿐.  
내부에서는 다른 클러스터들을 기다리는 중이다.  
(확인하고 싶으면 `start` 대신 `start-foreground`로 디버깅 가능)  
다른 클러스터들도 마저 켜줘야하니 바로 다음으로 진행한다.

나머지 컨테이너들도 내부로 진입하여 반복해준다.  
% `docker attach kafka-manual-zookeeper-cont-02`  
&#35; `/zookeeper/bin/zkServer.sh start`  

% `docker attach kafka-manual-zookeeper-cont-03`  
&#35; `/zookeeper/bin/zkServer.sh start`  
<br><br>

그 후, 각 클러스터 내부(`kafka-manual-zookeeper-cont-01`,`kafka-manual-zookeeper-cont-02`,`kafka-manual-zookeeper-cont-03`)에서 아래 명령어로 상태 확인.  
&#35; `/zookeeper/bin/zkServer.sh status`  
그러면  
```
ZooKeeper JMX enabled by default
Using config: /zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: leader 혹은 follower
```
로 잘 실행 중이며 `leader`나 `follower`로 결정된 것을 알 수 있다.  
<br><br>

이제 kafka 컨테이너 내부(`kafka-manual-cont-01`,`kafka-manual-cont-02`,`kafka-manual-cont-03`)에서  
&#35; `/kafka/bin/kafka-server-start.sh /kafka/config/server.properties`  
위 명령어를 셋 모두에서 실행하여, 카프카 역시 **멀티 브로커**로 실행해준다.  
<br><br>

% `docker attach kafka-manual-cont-producer`  
프로듀서용 컨테이너 내부에서  
&#35; `/kafka/bin/kafka-topics.sh --create --bootstrap-server kafka-01:9092,kafka-03:9092 --replication-factor 3 --partitions 3 --topic mytest`  
로 토픽을 하나 생성해본다.  
토픽이 생성되면  
&#35; `/kafka/bin/kafka-console-producer.sh --broker-list kafka-03:9092 --topic mytest`  
로 해당 토픽에 대해 원하는 문자열들을 몇 줄 작성해본다.  
<br><br>

% `docker attach kafka-manual-cont-consumer`  
컨슈머용 컨테이너 내부에서  
&#35; `/kafka/bin/kafka-console-consumer.sh --bootstrap-server kafka-01:9092,kafka-02:9092 --topic mytest --from-beginning`  
를 통해 메시지를 받아본다.
<br><br>
- - -
`FROM openjdk` 도커 이미지로부터  
ZooKeeper나 Kafka에 대한 환경변수 설정 없이  
기본 apache 파일들(`apache-zookeeper-3.8.0-bin`,`kafka_2.13-3.3.1`)과 설정 파일들(`zoo.cfg`,`myid`,`server.properties`)만으로  
멀티 클러스터, 멀티 브로커 예시를 준비했다.  
기본적인 설정을 이해했다면 이후엔 마음껏 바꿔가며 연구해보시길.
<br><br>
