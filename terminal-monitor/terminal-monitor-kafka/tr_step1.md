## Go Installation

Öncelikle kullanacağımız araçları kurarak başlayalım


```sh
cd ~
```
```sh
curl -OL https://golang.org/dl/go1.16.7.linux-amd64.tar.gz
```
```sh
sha256sum go1.16.7.linux-amd64.tar.gz
```
Çıktı şu şekilde olmalı: <b> 7fe7a73f55ba3e2285da36f8b085e5c0159e9564ef5f63ee0ed6b818ade8ef04  go1.16.7.linux-amd64.tar.gz </b>

```sh
sudo tar -C /usr/local -xvf go1.16.7.linux-amd64.tar.gz
```

Go binary'lerinin yerini .bashrc dosyasına ekleyelim. File -> open file ile show hidden files'ı göstererek yaparabilirsiniz ya da:

```sh
vim ~/.bashrc
```

bu kodu .bashrc dosyasına ekleyin
```sh
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export GOCACHE=$HOME/.cache/go-build
```

kaydedip çıkış yapın. esc + 
```sh
:wq
```

```sh
source ~/.bashrc
```

Go'nun versiyonunu kontrol edin

```sh
go version
```

  
## kafka installation

kafka ve kafka-ui'ı sisteme entegre edelim

```sh
touch docker-compose.yml
```

oluşturduğumuz docker-compose.yml dosyası içine aşağıdaki kodu yapıştırın

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

Sol taraftaki port simgesinden 8080 portuna bağlanarak kafka-ui'ı görünteleyebilirsiniz. "Configure new cluster" butonuna tıklayıp cluster name'e istediğiniz ismi, bootstrap servers'a kafka:9092 yazarak cluster'a bağlanabilirsiniz.

## topic creations.

kafka-ui üzerinden veya kafka container'ına bağlanarak 2 adet topic oluşturacağız.

kafka containerının terminaline bağlanacağız ve 2 adet topic oluşturacağız.

```sh
docker exec -it kafka-container bash
```
```sh
kafka-topics --create --topic error-topic --bootstrap-server localhost:9092
```
```sh
kafka-topics --create --topic tip-topic --bootstrap-server localhost:9092
```
exit yazarak container terminalinden çıkabilirsiniz.

Gerekli araçları yükledeğimize göre biraz kod yazmaya başlayabiliriz.