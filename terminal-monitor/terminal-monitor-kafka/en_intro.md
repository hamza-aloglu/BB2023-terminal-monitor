Overview
We will create a system that monitors errors occurring in users' terminals when used with Kafka and returns hints after a certain amount of errors.

<img width="1110" alt="Screenshot 2023-11-19 at 18 10 29" src="https://github.com/hamza-aloglu/BB2023-terminal-monitor/assets/74200100/68f72209-ceb6-47aa-8669-ce89f08aa34e">

The error-producer.sh file running in the background on the user's machine will send terminal errors to the error-topic. The error-tip-manager.go program will evaluate the information coming from the error-topic for each user separately and generate hint events, sending them to the tip-topic after a certain error threshold. In the background on each machine, the tip-consumer.go will listen to the tip-topic and write its relevant hints to the tips.txt file on the machine.

Let's begin the installation of the application.