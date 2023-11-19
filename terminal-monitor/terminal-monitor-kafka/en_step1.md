## Go Installation

Let's start by installing the necessary tools.

```sh
cd ~
```
```sh
curl -OL https://golang.org/dl/go1.16.7.linux-amd64.tar.gz
```
```sh
sha256sum go1.16.7.linux-amd64.tar.gz
```

The output should be: <b> 7fe7a73f55ba3e2285da36f8b085e5c0159e9564ef5f63ee0ed6b818ade8ef04 go1.16.7.linux-amd64.tar.gz </b>

```sh
sudo tar -C /usr/local -xvf go1.16.7.linux-amd64.tar.gz
```

Add the location of Go binaries to the .bashrc file. You can do this by showing hidden files with File -> Open File or:

```sh
vim ~/.bashrc
```

```sh
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export GOCACHE=$HOME/.cache/go-build
```

Check the version of Go:

```sh
go version
```

## kafka installation

Integrate Kafka and Kafka UI into the system:

```sh
touch docker-compose.yml
```

Paste the following code into the docker-compose.yml file:

```sh
version: "3"
services:
  zookeeper:
    container_name: zookeeper-container
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 22181:2181
    networks:
      - kafka-net
  kafka-ui:
    image: provectuslabs/kafka-ui
    container_name: kafka-ui-container
    ports:
      - 8080:8080
    environment:
      - DYNAMIC_CONFIG_ENABLED=true
    networks:
      - kafka-net
  kafka:
    container_name: kafka-container
    image: confluentinc/cp-kafka:latest
    user: root
    depends_on:
      - zookeeper
    ports:
      - 29092:29092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    volumes:
      - ./producer.sh:/home/appuser/producer.sh
    networks:
      - kafka-net
networks:
  kafka-net:
    driver: bridge
```

```sh
docker-compose up -d
```

Connect to the Kafka UI by navigating to port 8080 from the left-side port icon. Click "Configure new cluster," enter your desired cluster name, and input kafka:9092 for bootstrap servers to connect to the cluster.

## topic creations

We will create two topics either through Kafka UI or by connecting to the Kafka container's terminal.

Connect to the Kafka container's terminal and create two topics:

```sh
docker exec -it kafka-container bash
```
```sh
kafka-topics --create --topic error-topic --bootstrap-server localhost:9092
```
```sh
kafka-topics --create --topic tip-topic --bootstrap-server localhost:9092
```

Exit to leave the container terminal.

Now that we have installed the necessary tools, we can start writing some code.