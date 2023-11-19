## Overview

Kafka ile birlikte kullanıcıların terminallerinde yaşadıkları errorleri gözetleyen ve belli bir error miktarından sonra ipucu döndüren bir sistem oluşturacağız.

<img width="1139" alt="Screenshot 2023-11-19 at 05 03 01" src="https://github.com/hamza-aloglu/BB2023-terminal-monitor/assets/74200100/a02c49d2-e43f-4aa5-9084-6bc08ee570d5">

Kullanıcının makinesinde arka planda çalışan error-producer.sh dosyası terminalde yaşanan errorleri error-topic'e yollayacak. error-tip-manager.go programı error-topic'den gelen bilgileri her bir kullanıcıyı ayrı bir şekilde değerlendirerek ipucu eventleri üretecek ve tip-topic'e gönderecek. Her bir makinede yine arka planda çalışan tip-consumer.go tip-topic'i dinleyecek ve kendisine ait olan ipuçlarını makine üzerindeki tips.txt dosyasına yazdıracak.

Uygulamanın kurulumuna başlayalım.