# BB2023-monitor-terminal

## Overview

Kafka ile birlikte kullanıcıların terminallerinde yaşadıkları errorleri gözetleyen ve belli bir error miktarından sonra ipucu döndüren bir sistem oluşturacağız.

<img width="1139" alt="Screenshot 2023-11-19 at 05 03 01" src="https://github.com/hamza-aloglu/BB2023-terminal-monitor/assets/74200100/a02c49d2-e43f-4aa5-9084-6bc08ee570d5">

Kullanıcının makinesinde arka planda çalışan error-producer.sh dosyası terminalde yaşanan errorleri error-topic'e yollayacak. error-tip-manager.go programı error-topic'den gelen bilgileri her bir kullanıcıyı ayrı bir şekilde değerlendirerek ipucu eventleri üretecek ve tip-topic'e gönderecek. Her bir makinede yine arka planda çalışan tip-consumer.go tip-topic'i dinleyecek ve kendisine ait olan ipuçlarını makine üzerindeki tips.txt dosyasına yazdıracak.

Uygulamanın kurulumuna başlayalım.

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

## error-producer.sh

Kullanıcının makinesinde arka planda çalışacak olan uygulamaları kafka-client klasörü altında toplayalım.

```sh
mkdir kafka-client
```
```sh
cd kafka-client
```

error-producer.sh adı ile bir shell scripti yazacağız. Bu shell scripti terminalde çalıştırılan komutlardan eğer error üreten bir komut görürse bu komutu error-topic adlı topic'e publish edecek. Göndereceği verinin formatı şu şekilde olacak:
```json
{
  "vm": "<hostname of the machine>",
  "message": "<error output>",
  "step": "<in which step user took the error>"
}
```

error-producer'ı oluşturalım

```sh
touch error-producer.sh
```

aşağıdaki kodu error-producer.sh içine yapıştırın

```sh
DOCKER_CONTAINER_NAME="kafka-container"
TOPIC="error-topic"

get_hostname() {
    echo "$(hostname)"
}

send_to_kafka() {
    local hostname="$(get_hostname)"
    local message="$1"
    local json='{"VM": "'"${hostname}"'", "message": "'"${message}"'", "step": "'"$current_step"'"}'

    echo "$json" | docker exec -i $DOCKER_CONTAINER_NAME kafka-console-producer --bootstrap-server localhost:9092 --topic $TOPIC >/dev/null 2>&1
}

debug_handler() {
    command="$BASH_COMMAND"
    output=$(eval "$command" 2>&1)

    if [ $? -ne 0 ]; then
        # Command resulted in an error
        send_to_kafka "$command\n\n$output"
    fi
}

trap debug_handler DEBUG
```

bu kodun arka planda çalışması için ve her yeni açılan terminalde çalışabilmesi için .bashrc dosyasına şu komutu eklemeliyiz

```sh
source ~/kafka-client/error-producer.sh
```

Şu anda açık olan terminalde çalışması için bu komutu girin

```sh
source ~/.bashrc
```

Terminal üzerinde error üretecek bir komut yazın mesela rastgele harfler girip enter'a tıklayın daha sonra kafka-ui üzerinde error-topic altında error'unuzu görebileceksiniz.


## error-tip-manager.go

Uygulamanın ana mantığının çalıştığı yer bu Go uygulaması olacak. Bu Go dosyasında error-topic'i consume edeceğiz, error-topic üzerinde aynı hostname'e sahip error eventlerinden 3 veya daha fazla varsa tip-topic'e bir ipucu eventi gönderecek. Yani bu uygulama bir topic'i consume ederken aynı zamanda başka bir topic'e produce edecek.

Daha sonra kullanıcı makinesinde arka planda çalışan bir uygulama yazarak error-tip-manager.go tarafından gönderilen ipuçlarını alan bir consumer yazıp kullanıcının makinesinde göstereceğiz.

error-tip-manager.go kullanıcının makinesinde değil, kendi serverımızda çalışacak bir uygulama olacağı için bunu farklı bir klasör içinde oluşturalım

```sh
cd ~/
```
```sh
mkdir error-tip-manager
```
```sh
cd error-tip-manager/
```

go projesi oluşturalım ve içerisine kodu yapıştıralım


```sh
go mod init example/error-tip-manager
```
```sh
touch main.go
```

main.go dosyası içerisine aşağıdaki kodu yapıştırın

```sh
package main

import (
	"encoding/json"
	"fmt"
	"github.com/confluentinc/confluent-kafka-go/kafka"
	"io"
	"os"
	"os/signal"
	"syscall"
)

type TipEvent struct {
	VM  string `json:"vm"`
	Tip string `json:"message"`
}

type ErrorEvent struct {
	VM      string `json:"vm"`
	Message string `json:"message"`
	Step    string `json:"step"`
}

func main() {
	file, err := os.Open("tips.json")
	if err != nil {
		fmt.Println("Error opening file:", err)
		return
	}
	defer file.Close()

	fileContent, err := io.ReadAll(file)
	if err != nil {
		fmt.Println("Error reading file:", err)
		return
	}

	var stepTipMap map[string]string

	err = json.Unmarshal(fileContent, &stepTipMap)
	if err != nil {
		fmt.Println("Error unmarshalling JSON:", err)
		return
	}

	config := &kafka.ConfigMap{
		"bootstrap.servers": "127.0.0.1:29092",
		"group.id":          "error-consumer-group",
	}
	consumer, err := kafka.NewConsumer(config)
	if err != nil {
		fmt.Printf("Error creating consumer: %v\n", err)
		os.Exit(1)
	}

	topics := []string{"error-topic"}
	err = consumer.SubscribeTopics(topics, nil)
	if err != nil {
		fmt.Printf("Error subscribing to topics: %v\n", err)
		os.Exit(1)
	}
	defer consumer.Close()

	// Message count map for each VM
	messageCount := make(map[string]int)

	// Listen events
	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)
	run := true
	for run == true {
		select {
		case sig := <-sigchan:
			fmt.Printf("Caught signal %v: terminating\n", sig)
			run = false

		default:
			ev := consumer.Poll(100)

			switch e := ev.(type) {
			case *kafka.Message:
				var errorEvent ErrorEvent
				err := json.Unmarshal(e.Value, &errorEvent)
				if err != nil {
					fmt.Printf("Error decoding message: %v\n", err)
				} else {
					// Increment message count for the VM
					messageCount[errorEvent.VM]++

					if messageCount[errorEvent.VM] >= 3 {
						// Send an errorEvent to the "tips" topic
						tipsEvent := TipEvent{
							VM:  errorEvent.VM,
							Tip: stepTipMap[errorEvent.Step],
						}
						sendToTipsTopic(tipsEvent)
						// Reset the message count for the VM
						messageCount[errorEvent.VM] = 0
					}

					fmt.Printf("Received message from VM %s: %s\n", errorEvent.VM, errorEvent.Message)
				}

			case kafka.Error:
				fmt.Printf("Error: %v\n", e)

			default:
				// Ignore other events
			}
		}
	}
}

func sendToTipsTopic(event TipEvent) {
	config := &kafka.ConfigMap{
		"bootstrap.servers": "localhost:29092",
	}

	producer, err := kafka.NewProducer(config)
	if err != nil {
		fmt.Printf("Error creating producer: %v\n", err)
		return
	}
	defer producer.Close()

	topic := "tip-topic"
	value, err := json.Marshal(event)
	if err != nil {
		fmt.Printf("Error encoding event: %v\n", err)
		return
	}

	// Produce the message to the "tips" topic
	err = producer.Produce(&kafka.Message{
		TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
		Value:          value,
	}, nil)

	if err != nil {
		fmt.Printf("Error producing message: %v\n", err)
	}
}

```

Go kodu içerisinde topiclerdeki veri formatını bulabilirsiniz
```sh
type TipEvent struct {
	VM  string `json:"vm"`
	Tip string `json:"message"`
}

type ErrorEvent struct {
	VM      string `json:"vm"`
	Message string `json:"message"`
	Step    string `json:"step"`
}
```
Kullanıcının hangi adımda olduğuna göre ona farklı ipuçları vermek için "tips.json" adında bir dosya oluşturup içerisine ipuçlarını yazmalıyız.

```sh
touch tips.json
```
örnek olması açısından aşağıdaki kodu tips.json içerisine yapıştırabilirsiniz
```sh
{
  "step1": "try to use git merge on both branches",
  "step2": "research optional flags for git hash object command"
}
```

go uygulamasını build edip çalıştıralım

```sh
go get github.com/confluentinc/confluent-kafka-go/kafka
```
```sh
go build
```
```sh
./error-tip-manager
```

yeni bir terminal açıp gelecek adımlara oradan devam edin.

## tip-consumer.go

Gelen ipuçlarının kullanıcının makinesinde görüntülenebilmesi için bir consumer oluşturacağız. Bu consumer kendisine ait olan ipuçlarını tips.txt adlı dosyaya aktaracak. Bu kullanıcının bilgisayarında çalışacağı için kafka-client klasörüne gidelim.

```sh
cd ~/kafka-client
```

```sh
go mod init example/kafka-client
```

```sh
touch tip-consumer.go
```

Yukarıda anlattığım işlevi yerine getiren go kodunu tip-consumer.go içerisine yapıştıralım
```sh
package main

import (
	"encoding/json"
	"fmt"
	"github.com/confluentinc/confluent-kafka-go/kafka"
	"os"
	"os/signal"
	"syscall"
)

type TipEvent struct {
	VM  string `json:"vm"`
	Tip string `json:"message"`
}

func main() {
	config := &kafka.ConfigMap{
		"bootstrap.servers": "127.0.0.1:29092",
		"group.id":          "tip-consumer-group",
		"auto.offset.reset": "earliest",
	}
	consumer, err := kafka.NewConsumer(config)
	if err != nil {
		fmt.Printf("Error creating consumer: %v\n", err)
		os.Exit(1)
	}

	topics := []string{"tip-topic"}
	err = consumer.SubscribeTopics(topics, nil)
	if err != nil {
		fmt.Printf("Error subscribing to topics: %v\n", err)
		os.Exit(1)
	}
	defer consumer.Close()

	filePath := "/root/workspace/tips.txt"

	sigchan := make(chan os.Signal, 1)
	signal.Notify(sigchan, syscall.SIGINT, syscall.SIGTERM)
	run := true
	for run == true {
		select {
		case sig := <-sigchan:
			fmt.Printf("Caught signal %v: terminating\n", sig)
			run = false

		default:
			ev := consumer.Poll(100) // Set a timeout for polling, e.g., 100 milliseconds

			switch e := ev.(type) {
			case *kafka.Message:
				var tipEvent TipEvent
				err := json.Unmarshal(e.Value, &tipEvent)
				if err != nil {
					fmt.Printf("Error decoding message: %v\n", err)
				} else {
					hostname, _ := os.Hostname()
					if hostname == tipEvent.VM {

						file, err := os.OpenFile(filePath, os.O_WRONLY|os.O_APPEND|os.O_CREATE, 0666)
						if err != nil {
							if os.IsNotExist(err) {
								file, err = os.Create(filePath)
								if err != nil {
									panic(err)
								}
							} else {
								panic(err)
							}
						}
						defer file.Close()

						// Assuming tipEvent.Tip is a string, convert it to []byte
						tipData := []byte(tipEvent.Tip + "  |  \n")

						// Append data to the file
						_, err = file.Write(tipData)
						if err != nil {
							// Handle the error
							panic(err)
						}
					}
				}

			case kafka.Error:
				fmt.Printf("Error: %v\n", e)

			default:
				// Ignore other events
			}
		}
	}
}
```

```sh
go get github.com/confluentinc/confluent-kafka-go/kafka
```

```sh
go build
```
```sh
./kafka-client
```

## Test

Yeni bir terminal açıp uygulamamız çalışıyor mu test edelim.

Öncelikle, terminalden kullanıcının hangi adımda olduğunu anlamanın bir yöntemi olmadığı için hangi adımda olduğumuzu belirtmemiz gerekiyor.

```sh
export current_step=step1
```

3 tane error çıkartacak rastgele komutlar çalıştırın. Rastgele harflere basıp enter tuşuna basabilirsiniz.

```sh
cat ~/workspace/tips.txt
```

ipucunun geldiğini göreceksiniz.

Aynısını step2 için yapalım.
```sh
export current_step=step2
```

3 tane rastgele error komutu çalıştırın.

```sh
cat ~/workspace/tips.txt
```

Son 2 satırda şöyle bir çıktı gördüyseniz işlem tamamdır:
```sh
try to use git merge on both branches  |  
research optional flags for git hash object command  |
```






