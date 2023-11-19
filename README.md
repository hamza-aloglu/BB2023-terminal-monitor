# BB2023-monitor-terminal

## Overview

Kafka ile birlikte kullanıcıların terminallerinde yaşadıkları errorleri gözetleyen ve belli bir error miktarından sonra ipucu döndüren bir sistem oluşturacağız.

<img width="1139" alt="Screenshot 2023-11-19 at 05 03 01" src="https://github.com/hamza-aloglu/BB2023-terminal-monitor/assets/74200100/a02c49d2-e43f-4aa5-9084-6bc08ee570d5">

Kullanıcının makinesinde arka planda çalışan error-producer.sh dosyası terminalde yaşanan errorleri error-topic'e yollayacak. error-tip-manager.go programı error-topic'den gelen bilgileri her bir kullanıcıyı ayrı bir şekilde değerlendirerek ipucu eventleri üretecek ve tip-topic'e gönderecek. Her bir makinede yine arka planda çalışan tip-consumer.go tip-topic'i dinleyecek ve kendisine ait olan ipucularını makine üzerindeki tips.txt dosyasına yazdıracak.

Uygulamanın kurulumuna başlayalım.

## go installation,

```sh
something...
```

cd ~

curl -OL https://golang.org/dl/go1.16.7.linux-amd64.tar.gz
sha256sum go1.16.7.linux-amd64.tar.gz
Ouput should look like this: 7fe7a73f55ba3e2285da36f8b085e5c0159e9564ef5f63ee0ed6b818ade8ef04  go1.16.7.linux-amd64.tar.gz
sudo tar -C /usr/local -xvf go1.16.7.linux-amd64.tar.gz

vim ~/.bashrc
export PATH=$PATH:/usr/local/go/bin
export GOPATH=$HOME/go
export GOCACHE=$HOME/.cache/go-build
source ~/.bashrc

check: go version


  
## kafka installation,
touch docker-compose.yml
<the compose code>
docker-compose up -d
open kafka-ui on port 8080, configure broker, serverurl kafka

## topic creations.
docker exec kafka,
kafka-topics --create --topic error-topic --bootstrap-server localhost:9092
kafka-topics --create --topic tip-topic --bootstrap-server localhost:9092


## error-producer.sh
mkdir kafka-client
cd kafka-client
touch error-producer.sh:
source ./error-producer.sh (write this into .bashrc)

## error-tip-manager.go
cd ~/
mkdir error-tip-manager
cd error-tip-manager/
go mod init example/error-tip-manager
touch main.go
<the code>
go get github.com/confluentinc/confluent-kafka-go/kafka
create tips.json
go build
./error-tip-manager

## kafka-client (?)
cd ~/root/workspace/
go mod init example/kafka-client
touch tip-consumer.go
go get github.com/confluentinc/confluent-kafka-go/kafka
go build
go run kafka-client
