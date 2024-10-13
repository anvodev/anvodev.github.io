---
layout: post
title: 'Getting Start with Kafka using Docker'
date: 2024-04-10
categories: dev
---

In this article, we will use Kafka via docker and try producing and consuming messages. We will learn some keywords and basic tools in Kafka.

Firstly, let's create a new folder named `kafka` (or whatever name of your project), and go to that folder

```
mkdir kafka && cd kafka
```

Inside the `kafka` folder, we create new `docker-compose.yml` file with the content below:

```
version: '3.8'

services:
  kafka: # service name, can change to any name
    image: 'bitnami/kafka:3.6.2' # docker image from https://hub.docker.com/r/bitnami/kafka
    environment:
      - KAFKA_CFG_NODE_ID=0
      - KAFKA_CFG_PROCESS_ROLES=controller,broker
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
      - KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER

```

Run Kafka server

```
docker-compose up -d
```

We need the container name to access into it. Check the `NAMES` column when running:

```
docker ps
---
CONTAINER ID   IMAGE                      COMMAND                  CREATED        STATUS          PORTS           NAMES
f9258b058804   bitnami/kafka:3.6.2        "/opt/bitnami/script…"   24 hours ago   Up 39 minutes   9092/tcp        kafka_kafka_1
```

Access into docker container command line (In my case in `kafka_kafka_1`, please change to your service name if mismatch)

```
docker exec -it kafka_kafka_1 bash
```

Now you are in the container and can use a set of tools below:

- `kafka-topics.sh`: Manage topics
- `kafka-console-producer.sh`: Manage producer
- `kafka-console-consumer.sh`: Manage consumer

Kafka treat message as "event". Events and grouped into a category called "topic".
Assume we are creating a chat application. Let's create a new topic called "chat-message" which is a stream of message records, as well as create a new topic called "user-presence" to manage the online status of users. We will use the server kafka:9092 mentioned in docker-compose configuration

```
kafka-topics.sh --create --topic chat-message --bootstrap-server kafka:9092
kafka-topics.sh --create --topic user-presence --bootstrap-server kafka:9092
```

We can check how many topics we have by using:

```
kafka-topics.sh --list --bootstrap-server kafka:9092
---
__consumer_offsets
chat-message
user-presence
```

And view detail each topic:

```
kafka-topics.sh --describe --topic chat-message --bootstrap-server kafka:9092
---
Topic: chat-message	TopicId: 0XIXOlZ-TAWqEBlBUP7WAA	PartitionCount: 1	ReplicationFactor: 1	Configs:
	Topic: chat-message	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
```

Now we can use 2 separate terminals to set up the producer and consumer, and see how the consumer get message from the producer in real time.

In the first terminal, let's use the tool `kafka-console-consumer.sh` to setup a consumer to waiting for the messages in the topic `chat-message` to consume:

```
kafka-console-consumer.sh --topic chat-message --from-beginning --bootstrap-server kafka:9092
```

Open the second terminal and use the tool `kafka-console-producer.sh` to setup the producer by running the command:

```
kafka-console-producer.sh --topic chat-message --bootstrap-server kafka:9092
```

The `kafka-console-producer.sh` will waiting for our inputs. Let's try pasting 3 events below and hit enter:

```
Event: {"message_id": 1, "sender_id": "user1", "receiver_id": "user2", "content": "Hello, how are you?", "timestamp": "2024-04-10T09:00:00"}
Event: {"message_id": 2, "sender_id": "user2", "receiver_id": "user1", "content": "I'm good, thanks! How about you?", "timestamp": "2024-04-10T09:01:00"}
Event3: some random words...
```

Note that the message format is plaintext, in the example above we use both plain text and some json-like format to show that the format is not strict. The consumer will need to have to logic to decode the plaintext to some meaningful messages.

Now we should see events appear in the consumer terminal

That is the very basic setup of Kafka using docker.

**Bonus**:
If you don't want to access the container, in the host machine you can replace the `bash` with the kafka tools like `kafka-topics.sh` to run the Kafka command outside of container like this:

```
docker exec -it kafka_kafka_1 kafka-topics.sh --list --bootstrap-server kafka:9092
```
