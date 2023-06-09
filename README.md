# Kafka Streams

- Created a maven project
- Add the dependencies kafka-streams, kafka-clients,slf4j
- Create a WordCountApp java class file
- Add the following code

- 

```java
package com.shilpalearning.wordcount;

import java.util.Arrays;
import java.util.Properties;

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.Materialized;
import org.apache.kafka.streams.kstream.Produced;

public class WordCountApp {

	public static void main(String[] args) {
		Properties props = new Properties();
		props.put(StreamsConfig.APPLICATION_ID_CONFIG,"word-count-app");
		props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"localhost:29092");
		props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,Serdes.String().getClass());
		props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());
		props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,"earliest");
		props.put(StreamsConfig.CACHE_MAX_BYTES_BUFFERING_CONFIG,"0");

		StreamsBuilder streamsBuilder = new StreamsBuilder();
		//Build topology
		streamsBuilder.<String, String>stream("sentences")
						.flatMapValues((readOnlyKey, value) -> Arrays.asList(value.toLowerCase().split(" ")))
						.groupBy((key,value)->value)
						.count(Materialized.with(Serdes.String(), Serdes.Long()))
						.toStream()
						.to("word-count", Produced.with(Serdes.String(),Serdes.Long()));
		KafkaStreams kafkaStreams = new KafkaStreams(streamsBuilder.build(), props);
		
		kafkaStreams.start();
		Runtime.getRuntime().addShutdownHook(new Thread(kafkaStreams::close));
	}
}
```
First add the properties related to bootstrap servers, serialization, offset,

Create a StreamsBuilder object which belongs to kafka streams API

we are using the stream of the topic sentences and converting to flatmap then grouping by keys so that it is grouped with the same word as key and we are couting the words and writing to word-count topic which is read by the consumer

We are starting the streams on a thread

- Create a docker compose file which uses confluent images for kafka, zoo keeper and creating topics
- 

```java
version: '3'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:6.0.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "32181:32181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 32181
      ZOOKEEPER_TICK_TIME: 2000

  kafka:
    image: confluentinc/cp-enterprise-kafka:6.0.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:32181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
  kafka-create-topics:
    image: confluentinc/cp-enterprise-kafka:6.0.0
    depends_on:
      - kafka
    hostname: kafka-create-topics
    command: ["bash","./create-topics.sh"]
    working_dir: /scripts
    volumes: 
      - ./scripts:/scripts
```

Create scripts folder in the root with [create-topics.sh](http://create-topics.sh) file which has the commands to create the topics and is attached to the volume of the docker

In create-scripts.sh

```bash
echo 'Waiting for Kafka to come online..' && \
cub kafka-ready -b kafka:9092 1 20 && \
kafka-topics --bootstrap-server kafka:9092 --topic sentences --replication-factor 1 --partitions 1 --create && \
kafka-topics --bootstrap-server kafka:9092 --topic word-count --replication-factor 1 --partitions 1 --create && \
sleep infinity
```

when you have docker running and the docker-compose up command is executed, it creates the 2 topics with name sentences and word-count with replication factor 1 and partition 1 once the kafka is ready

you can open the terminal for consumer and producer

```bash
sh-4.4$ kafka-console-producer --topic sentences --bootstrap-server localhost:9092
```

```bash
sh-4.4$  kafka-console-consumer --topic word-count --bootstrap-server localhost:9092 --from-beginning --property print.key=true --property key.seperators=" : " --key-deserializer "org.apache.kafka.common.seria
lization.StringDeserializer" --value-deserializer "org.apache.kafka.common.serialization.LongDeserializer"
```

where producer publishes to the topic sentences and consumer consumes from the topic word-count.

The output would be like

Producer - 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dc70dda6-2880-4fb8-98c8-e1aa8cdb90cb/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ec77d224-dd21-43cd-abbf-560e9178bee6/Untitled.png)

Consumer we get this output
