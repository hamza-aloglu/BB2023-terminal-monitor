## error-producer.sh

Let's gather applications running in the background on the user's machine under the kafka-client folder.

```sh
mkdir kafka-client
```
```sh
cd kafka-client
```

We will write a shell script named error-producer.sh. This shell script, when executed in the terminal, will publish a command to the error-topic if it encounters a command that produces an error. The format of the data it sends will be as follows:

```json
{
  "vm": "<hostname of the machine>",
  "message": "<error output>",
  "step": "<in which step user took the error>"
}
```

Create the error-producer:

```sh
touch error-producer.sh
```

Paste the following code into error-producer.sh:

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

To run this code in the background and make it work in every new terminal, we need to add the following command to the .bashrc file:

```sh
source ~/kafka-client/error-producer.sh
```

```sh
source ~/.bashrc
```

Enter a command in the terminal that produces an error, for example, enter random letters and press Enter. Later, you can see your error under the error-topic in Kafka-UI.

## error-tip-manager.go

The main logic of the application will be in this Go application. In this Go file, we will consume the error-topic, and if there are 3 or more error events with the same hostname on the error-topic, it will send a hint event to the tip-topic. In other words, this application will consume one topic while producing to another.

Since error-tip-manager.go will be an application running on our server, let's create it in a different folder.

```sh
cd ~/
```
```sh
mkdir error-tip-manager
```
```sh
cd error-tip-manager/
```

Create a Go project and paste the code into it:

```sh
go mod init example/error-tip-manager
```
```sh
touch main.go
```

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

Create a file named "tips.json" and add hints for different steps. For example:

```sh
touch tips.json
```

```sh
{
  "step1": "try to use git merge on both branches",
  "step2": "research optional flags for git hash object command"
}
```

Build and run the Go application:

```sh
go get github.com/confluentinc/confluent-kafka-go/kafka
```
```sh
go build
```
```sh
./error-tip-manager
```

Open a new terminal and continue with the next steps.

## tip-consumer.go

We will create a consumer to display the received hints on the user's machine. This consumer will write its hints to a file named tips.txt. Since this consumer will run on the user's computer, let's go to the kafka-client folder.

```sh
cd ~/kafka-client
```

```sh
go mod init example/kafka-client
```

```sh
touch tip-consumer.go
```

Paste the provided Go code into tip-consumer.go.

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