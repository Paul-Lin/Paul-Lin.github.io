### Step1:Start Service
> sh bin/zookeeper-server-start.sh config/zookeeper.properties &
> sh bin/kafka-server-start.sh config/server.properties &

### Step2:Create topic
> sh bin/kafka-topics.sh  --zookeeper localhost:2181 \
--replication-factor 1 --partitions 2 --create --topic test

#### Step2.1:Check topic by list command
> sh bin/kafka-topics.sh  --zookeeper localhost:2181 --list 

### Step3:Sent Message
> sh bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

### Step4:Start Consumer
> sh bin/kafka-console-consumer.sh --zookeeper locahost:2181  \
--from-beginning --topic test

***
## How to develop in Kafka 
### Step1:Configure Kakfa connected properties
``` java
public final class MoonKafkaProperties {
	public static final String TOPIC = "topic1";
    public static final String KAFKA_SERVER_URL = "localhost";
    public static final int KAFKA_SERVER_PORT = 9092;
    public static final int KAFKA_PRODUCER_BUFFER_SIZE = 64 * 1024;
    public static final int CONNECTION_TIMEOUT = 100000;
    public static final String TOPIC2 = "topic2";
    public static final String TOPIC3 = "topic3";
    public static final String CLIENT_ID = "SimpleConsumerDemoClient";

	private MoonKafkaProperties(){}
}

```
### Step2:Create Kafka producer
``` java
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

public class MoonSimpleProducer extends Thread {
	private final KafkaProducer<Integer, String> producer;
    private final String topic;
    private final Boolean isAsync;

    public MoonSimpleProducer(String topic, Boolean isAsync) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("client.id", "DemoProducer");
        props.put("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer<>(props);
        this.topic = topic;
        this.isAsync = isAsync;
    }

    public void run() {
        int messageNo = 1;
        while (true) {
            String messageStr = "Message_" + messageNo;
            long startTime = System.currentTimeMillis();
            if (isAsync) { // Send asynchronously
                producer.send(new ProducerRecord<>(topic,
                    messageNo,
                    messageStr), new MoonCallBack(startTime, messageNo, messageStr));
            } else { // Send synchronously
                try {
                    producer.send(new ProducerRecord<>(topic,
                        messageNo,
                        messageStr)).get();
                    System.out.println("Sent message: (" + messageNo + ", " + messageStr + ")");
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
            ++messageNo;
        }
    }
}

class MoonCallBack implements Callback {

    private final long startTime;
    private final int key;
    private final String message;

    public MoonCallBack(long startTime, int key, String message) {
        this.startTime = startTime;
        this.key = key;
        this.message = message;
    }

    /**
     * A callback method the user can implement to provide asynchronous handling of request completion. This method will
     * be called when the record sent to the server has been acknowledged. Exactly one of the arguments will be
     * non-null.
     *
     * @param metadata  The metadata for the record that was sent (i.e. the partition and offset). Null if an error
     *                  occurred.
     * @param exception The exception thrown during processing of this record. Null if no error occurred.
     */
    public void onCompletion(RecordMetadata metadata, Exception exception) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        if (metadata != null) {
            System.out.println(
                "message(" + key + ", " + message + ") sent to partition(" + metadata.partition() +
                    "), " +
                    "offset(" + metadata.offset() + ") in " + elapsedTime + " ms");
        } else {
            exception.printStackTrace();
        }
    }
}
```
### Step3:Create Kafka consumer
``` java
import kafka.utils.ShutdownableThread;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Collections;
import java.util.Properties;

public class MoonSimpleConsumer extends ShutdownableThread{
	 private final KafkaConsumer<Integer, String> consumer;
	    private final String topic;

	    public MoonSimpleConsumer(String topic) {
	        super("KafkaConsumerExample", false);
	        Properties props = new Properties();
	        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
	        props.put(ConsumerConfig.GROUP_ID_CONFIG, "DemoConsumer");
	        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
	        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000");
	        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000");
	        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.IntegerDeserializer");
	        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");

	        consumer = new KafkaConsumer<>(props);
	        this.topic = topic;
	    }

	    @Override
	    public void doWork() {
	        consumer.subscribe(Collections.singletonList(this.topic));
	        ConsumerRecords<Integer, String> records = consumer.poll(1000);
	        for (ConsumerRecord<Integer, String> record : records) {
	            System.out.println("Received message: (" + record.key() + ", " + record.value() + ") at offset " + record.offset());
	        }
	    }

	    @Override
	    public String name() {
	        return null;
	    }

	    @Override
	    public boolean isInterruptible() {
	        return false;
	    }
}

```