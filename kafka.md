## Container Setup
Get the Docker image:
```
docker pull apache/kafka:4.1.1
```

Start the Kafka Docker container:
```
docker run -d --name=kafka -p 9092:9092 apache/kafka
```

Verify the cluster is up and running and get its cluster ID:
```
docker exec -ti kafka /opt/kafka/bin/kafka-cluster.sh cluster-id --bootstrap-server :9092
```

Next, we create the input topic named streams-plaintext-input and the output topic named streams-wordcount-output:
```
docker exec -ti kafka /opt/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-plaintext-input
```
```
 docker exec -ti kafka /opt/kafka/bin/kafka-topics.sh --create \
    --bootstrap-server localhost:9092 \
    --replication-factor 1 \
    --partitions 1 \
    --topic streams-wordcount-output \
    --config cleanup.policy=compact
```

Start the WordCount Application
```
mvn clean package
mvn exec:java -Dexec.mainClass=myapps.WordCoun
```

Now we can start the console producer in a separate terminal to write some input data to this topic:
```
docker exec -ti kafka /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic streams-plaintext-input
```

and inspect the output of the WordCount demo application by reading from its output topic with the console consumer in a separate terminal:
```
docker exec -ti kafka /opt/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic streams-wordcount-output \
    --from-beginning \
    --property print.key=true \
    --property print.value=true \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

with compose:
```
docker-compose up -d
```

## Kafka Listeners

### 1. What is a listener?

A **listener** is an **address + port** where a server (like Kafka) listens for incoming connections.

Think of it like:

> “If any client wants to talk to me, I am listening at this port.”

**Examples:**

* `HOST://0.0.0.0:9092`
* `DOCKER://0.0.0.0:9093`
* `CONTROLLER://localhost:9091`

Each one is like a *door* where Kafka accepts requests.

Kafka allows multiple listeners because:

* apps on the host machine need one address
* apps inside Docker need another
* internal components (like controllers) need another

Each listener can be **separately configured and advertised**.

---

### 🔒 2. What does “bind” mean?

To **bind** means:

> “What network interface and port should the server open and listen on?”

**Examples:**

* `0.0.0.0` → bind on **all** network interfaces
* `localhost` → bind only on the **loopback** interface
* `192.168.1.10` → bind on a **specific** interface

So:

```
HOST://0.0.0.0:9092
```

Means:

> “Kafka should listen on port **9092**, on **every interface** inside the container.”

---

### Internal listener vs Advertised listener

| Concept                       | Meaning                                |
| ----------------------------- | -------------------------------------- |
| **Internal (bound) listener** | Where Kafka actually listens           |
| **Advertised listener**       | What clients should use to reach Kafka |

---

### ✔ Real example from your config

Kafka **binds internally** to:

```
HOST://0.0.0.0:9092
DOCKER://0.0.0.0:9093
```

But **advertises**:

```
HOST://localhost:9092     ← what host apps should use
DOCKER://kafka:9093       ← what Docker apps should use
```

---

### Meaning:

| Purpose                   | Internal (bound) | Advertised (mapped) |
| ------------------------- | ---------------- | ------------------- |
| Host machine connects     | `0.0.0.0:9092`   | `localhost:9092`    |
| Docker container connects | `0.0.0.0:9093`   | `kafka:9093`        |

So in **advertised listeners**, “map” really means:

> “Tell clients that this listener should be reached at **this** public address.”

Kafka **internally** listens at `0.0.0.0:9092`,
but clients should **not** use that address — they should use:

```
localhost:9092
```

Kafka essentially **maps** the internal endpoint to a client-friendly external one.
