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