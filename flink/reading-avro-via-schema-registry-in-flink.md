## Reading Avro via Schema Registry in Flink

To deserialize, in pom:

```
<dependency>
    <groupId>io.confluent</groupId>
    <artifactId>kafka-avro-serializer</artifactId>
    <version>3.3.1</version>
</dependency>
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka_2.11</artifactId>
    <version>0.11.0.1</version>
</dependency>
```

Create deserialization class implementing Flink's `DeserializationSchema`:

```
package com.example.bigdata.serialization;

import io.confluent.kafka.schemaregistry.client.CachedSchemaRegistryClient;
import io.confluent.kafka.schemaregistry.client.SchemaRegistryClient;
import io.confluent.kafka.serializers.KafkaAvroDecoder;
import org.apache.flink.api.common.typeinfo.BasicTypeInfo;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.streaming.util.serialization.DeserializationSchema;

public class ConfluentAvroDeserializationSchema implements DeserializationSchema<String> {

    private final String schemaRegistryUrl;
    private KafkaAvroDecoder kafkaAvroDecoder;

    public ConfluentAvroDeserializationSchema(String schemaRegistryUrl) {
        this.schemaRegistryUrl = schemaRegistryUrl;
    }

    @Override
    public String deserialize(byte[] message) {
        if (kafkaAvroDecoder == null) {
            SchemaRegistryClient schemaRegistry = new CachedSchemaRegistryClient(this.schemaRegistryUrl, 1000);
            this.kafkaAvroDecoder = new KafkaAvroDecoder(schemaRegistry);
        }
        return this.kafkaAvroDecoder.fromBytes(message).toString();
    }

    @Override
    public boolean isEndOfStream(String nextElement) {
        return false;
    }

    @Override
    public TypeInformation<String> getProducedType() {
        return BasicTypeInfo.STRING_TYPE_INFO;
    }

}
```

Then create a Kafka consumer source and use the new deserializer class:

```
DataStream<String> input = env.addSource(
        new FlinkKafkaConsumer010<>(
                fileProperties.getProperty("source.topic"),
                new ConfluentAvroDeserializationSchema("http://localhost:8082"),
        consumerProperties), "kafka/flink avro testing source");
```



To write to a topic do the same thing in reverse:

```
package com.example.bigdata.serialization;

import io.confluent.kafka.schemaregistry.client.CachedSchemaRegistryClient;
import io.confluent.kafka.schemaregistry.client.SchemaRegistryClient;
import io.confluent.kafka.serializers.KafkaAvroEncoder;
import org.apache.flink.streaming.util.serialization.SerializationSchema;

public class ConfluentAvroSerializationSchema implements SerializationSchema<String> {

    private final String schemaRegistryUrl;
    private KafkaAvroEncoder kafkaAvroEncoder;

    public ConfluentAvroSerializationSchema(String schemaRegistryUrl) {
        this.schemaRegistryUrl = schemaRegistryUrl;
    }

    @Override
    public byte[] serialize(String message) {
        if (this.kafkaAvroEncoder == null) {
            SchemaRegistryClient schemaRegistryClient = new CachedSchemaRegistryClient(this.schemaRegistryUrl, 1000);
            this.kafkaAvroEncoder = new KafkaAvroEncoder(schemaRegistryClient);
        }
        return this.kafkaAvroEncoder.toBytes(message);
    }
}
```

And create the sink:

```
input.addSink(
        new FlinkKafkaProducer010<>(
                fileProperties.getProperty("sink.topic"),
                new ConfluentAvroSerializationSchema(schemaRegistryUrl),
                producerProperties)).name("kafka/flink avro testing sink");
```



