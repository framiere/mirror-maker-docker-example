# Objective
1. Setup 2 kafka clusters : source & destination
1. Create a topic on source with dummy data
1. Run mirror-maker 
1. Verify the topic is created on destination with the corresponding data

# Live action
[![sreencast](https://asciinema.org/a/FSMC1bbQ2kAtrCEHkY2KUQ5Pq.png)](https://asciinema.org/a/FSMC1bbQ2kAtrCEHkY2KUQ5Pq?autoplay=1)

# Kafka Clusters
* source cluster* kafka-source, zookeeper-source
* destination cluster : kafka-destination, kafka-destination

# Topic to mirror
* name : topic-to-mirror
* data : sequence of the 10000 numbers

```yml
  topic-creation:
    image: confluentinc/cp-kafka:4.0.0
    command: bash -c "cub kafka-ready -z zookeeper-source:2181 1 30 && kafka-topics --zookeeper zookeeper-source:2181 --create --topic topic-to-mirror --partitions 10 --replication-factor 1"
    depends_on:
      - zookeeper-source

  dummy-generation:
    image: confluentinc/cp-kafka:4.0.0
    command: bash -c "cub kafka-ready -z zookeeper-source:2181 1 30 && sleep 5 && seq 10000 | kafka-console-producer --broker-list kafka-source:9092 --topic topic-to-mirror"
    depends_on:
      - zookeeper-source
      - kafka-source
```

# Mirror maker
```yml
  mirror-maker:
    image: confluentinc/cp-kafka:4.0.0
    volumes:
      - ./consumer.cfg:/etc/consumer.cfg
      - ./producer.cfg:/etc/producer.cfg
    command: bash -c "cub kafka-ready -z zookeeper-source:2181 1 30 && cub kafka-ready -z zookeeper-destination:2181 1 30 && kafka-mirror-maker --consumer.config /etc/consumer.cfg --producer.config /etc/producer.cfg --whitelist topic-to-mirror --num.streams 1"
    depends_on:
      - kafka-source
      - kafka-destination
      - zookeeper-destination
 ```

## consumer.cfg
```properties
bootstrap.servers=kafka-source:9092
exclude.internal.topics=true
group.id=mirror-maker
auto.offset.reset=earliest
partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor
```

## producer.cfg
```properties
bootstrap.servers=kafka-destination:9092
acks=-1
batch.size=1
client.id=mirror_maker_producer
retries=2147483647
max.in.flight.requests.per.connection=1
```

# Commands

```
$ docker-compose up -d
$ docker-compose ps
```

Wait for Zookeeper, Kafka, and dummy data producer to startup

```
$ docker-compose exec kafka-source kafka-topics --zookeeper zookeeper-source:2181 --list
$ docker-compose exec kafka-source kafka-console-consumer --bootstrap-server localhost:9092 --topic topic-to-mirror --from-beginning
```

Ok, we have our topic in the source cluster ready 

```
$ docker-compose exec kafka-destination kafka-topics --zookeeper zookeeper-destination:2181 --list
$ docker-compose exec kafka-destination kafka-console-consumer --bootstrap-server localhost:9092 --topic topic-to-mirror --from-beginning
```

mirror-maker did replicate on the destination cluster

for demonstrating purposes use 
```
docker-compose exec kafka-source kafka-console-producer --broker-list localhost:9092 --topic topic-to-mirror
```
and start typing
