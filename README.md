docker-compose up -d
docker-compose ps
docker-compose logs -f mirror-maker
docker-compose exec kafka-source kafka-topics --zookeeper zookeeper-source:2181 --list
docker-compose exec kafka-source kafka-console-consumer --bootstrap-server localhost:9092 --topic topic-to-mirror --from-beginning
docker-compose exec kafka-destination kafka-console-consumer --bootstrap-server localhost:9092 --topic topic-to-mirror --from-beginning