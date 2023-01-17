# docker-kafka-example
Kafka with ZooKeeper Example for Docker Compose
<br><br>

프로젝트 클론 후, Dockerfile들의 COPY에 필요한 파일들을 준비한다.  

https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.8.0/apache-zookeeper-3.8.0-bin.tar.gz  
위 페이지에서 받은 파일의 압축을 풀고, zookeeper-3.8.0-bin 디렉터리의 이름을 zookeeper로 하여 함께 둔다.  
(./zookeeper/bin, ./zookeeper/conf, ...)

https://www.apache.org/dyn/closer.cgi?path=/kafka/3.3.1/kafka_2.13-3.3.1.tgz  
위 페이지에서 받은 파일의 압축을 풀고, kafka_2.13-3.3.1 디렉터리의 이름을 kafka로 하여 함께 둔다.  
(./kafka/bin, ./kafka/config, ...)  
<br><br>  

docker-compose version은 v.2.15.1 이상  
`docker compose up -d`  
로 빌드 및 실행한다.
<br><br>

`docker attach kafka-cont-01`  
내부로 진입하여  
`/zookeeper/bin/zkServer.sh start`  
그러면  
```
ZooKeeper JMX enabled by default
Using config: /zookeeper/bin/../conf/zoo.cfg  
Starting zookeeper ... FAILED TO START
```
FAILED TO START로 뜨지만, 멀티 클러스터 환경이라 클러스터 수가 부족하여 그런 것 뿐.  

나머지 컨테이너들도 내부로 진입하여 반복해준다.  
`docker attach kafka-cont-02`  
`/zookeeper/bin/zkServer.sh start`  
`docker attach kafka-cont-03`  
`/zookeeper/bin/zkServer.sh start`  
<br><br>

그 후, 각 클러스터 내부(`kafka-cont-01`,`kafka-cont-02`,`kafka-cont-03`)에서 아래 명령어로 상태 확인.  
`/zookeeper/bin/zkServer.sh status`  
그러면  
```
ZooKeeper JMX enabled by default
Using config: /zookeeper/bin/../conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: leader 혹은 follower
```
로 잘 실행 중이며 leader와 follower가 결정된 것을 알 수 있다.  
<br><br>

각 클러스터 내부(`kafka-cont-01`,`kafka-cont-02`,`kafka-cont-03`)에서  
`/kafka/bin/kafka-server-start.sh /kafka/config/server.properties`  
로 카프카 역시 실행해준다.  
<br><br>

`docker attach kafka-cont-producer`  
이 컨테이너 내부에서  
`/kafka/bin/kafka-topics.sh --create --bootstrap-server kafka1:9092,kafka2:9092,kafka3:9092 --replication-factor 3 --partitions 1 --topic mytest`  
로 토픽을 하나 생성해본다.  
토픽이 생성되면  
`/kafka/bin/kafka-console-producer.sh --broker-list kafka1:9092,kafka2:9092,kafka3:9092 --topic mytest`  
로 해당 토픽에 대해 원하는 문구들을 publish해본다.  
<br><br>

`docker attach kafka-cont-consumer`  
이 컨테이너는 ENTRYPOINT가 미리 mytest 토픽을 subscribe하도록 되어 있다.  
(`ENTRYPOINT [ "/kafka/bin/kafka-console-consumer.sh", "--bootstrap-server", "kafka1:9092,kafka2:9092,kafka3:9092", "--topic", "mytest", "--from-beginning" ]`)  
실행되어 있었으니, log를 통해 문구들을 확인할 수 있다.  
