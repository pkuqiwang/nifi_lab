# Contents
- [Lab 1](#lab-1)
  - Build lab environment
- [Lab 2](#lab-2)
  - Build a simple NiFi data flow
- [Lab 3](#lab-3) 
  - Using Nifi Templates
- [Lab 4](#lab-4) 
  - MiNifi and remote process group
- [Lab 5](#lab-5) 
  - Kafka basics
- [Lab 6](#lab-6)
  - Integrate Kafka with Nifi

## Goals of the Labs:
  - Setup sandbox lab environment	
  - Consume Meetup RSVP stream
  - Extract the JSON elements we are interested in
  - Split the JSON into smaller fragmentsand 
  - Create, save, upload Nifi template
  - Create Nifi flow from template
  - Send data to remote Nifi instance
  - Write data to Kafka with tool and Nifi
  - Consume data from Kafka with tool and Nifi
  - Write data to local disk

We will run through a series of labs and step by step to achieve all of the above goals

------------------

# Lab 1
## Build lab environment

### Goals:
   - Download and start Hortonworks HDF Sandbox
   - Access sandbox Ambari UI and Nifi UI
   - Remote to Sandbox console
   
## Get HDF Sandbox

Download the HDF [Sandbox](https://hortonworks.com/downloads/#sandbox) from Hortonworks website.

All the following instrucitons are based on the [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version of the Sandbox, if you use VMWare, you might need to make slight changes to some of the settings.

## Use the Sandbox

Once the Sandbox is started in VirtualBox, the start page will show you the start page link like [http://127.0.0.1:18888](http://127.0.0.1:18888)
![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab1-0.png)

### Connect to VM from Windows using Putty or Linux/MacOSX using ssh
	- host: 127.0.0.1
	- port: 12222
	- user: root
	- password: hadoop

### Reset Ambari password

- once you remote inside the sandbox, use following command to reset Ambari password
```
# Updates password
ambari-admin-password-reset
# If Ambari doesn't restart automatically, restart ambari service
ambari-agent restart
```

### Login to Ambari

- Access Ambari UI at [http://127.0.0.1:9080](http://127.0.0.1:9080)
	user: admin
	password: the one you set in previous step

### NiFi UI

- Access Nifi UI at [http://127.0.0.1:19090/nifi](http://127.0.0.1:19090/nifi)
- For more reference about Nifi, please go to Nifi [documents](https://nifi.apache.org/docs.html)

-----------------------------

# Lab 2

## Build Nifi Flow

### Goals:
   - Consume Meetup RSVP stream
   - Extract the JSON elements we are interested in
   - Split the JSON into smaller fragments
   - Use funnel as a temporary processor sink
   
## Consuming RSVP Data

To get started we need to consume the data from the Meetup RSVP stream, extract what we need, split the content and save it to a file:

Our final flow for this lab will look like the following:
![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab2.png)

1. Drag a Processor Group to the canvas and name it ```Nifi Lab```
	- double click the newly create Porcessor Group and create a new Processor Group called ```Lab 2```
	- double click ```Lab 2``` Processor Group and continue the following steps inside

2. Add a ConnectWebSocket processor to the cavas
	- Configure the WebSocket Client Controller Service. The WebSocket URI for the meetups is: ```ws://stream.meetup.com/2/rsvps```
	- Set WebSocket Client ID to your favorite number.
      	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab2-1.png)
	- Enable controller service by click the flash icon
      
3. Add an Update Attribute procesor
	- Configure it to have a custom property called ``` mime.type ``` with the value of ```application/json```
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab2-2.png)
    
4. Add an EvaluateJsonPath processor and configure it as shown below:
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab2-3.png)

    	- The properties to add are:
	```
	event.name	$.event.event_name  
	event.url	$.event.event_url    
	group.city	$.group.group_city    
	group.state	$.group.group_state    
	group.country	$.group.group_country    
	group.name	$.group.group_name    
	venue.lat	$.venue.lat    
	venue.lon	$.venue.lon    
	venue.name	$.venue.venue_name
	```
    
5. Add a SplitJson processor and configure the JsonPath Expression to be ```$.group.group_topics ```
  
6. Add a ReplaceText processor and configure the Search Value to be ```([{])([\S\s]+)([}])``` and the Replacement Value to be
```
{
	"event_name": "${event.name}",
	"event_url": "${event.url}",
	"venue" : {
		"lat": "${venue.lat}",
		"lon": "${venue.lon}",
		"name": "${venue.name}"
	},
	"group" : {
	  	"group_city" : "${group.city}",
	  	"group_country" : "${group.country}",
	  	"group_name" : "${group.name}",
	  	"group_state" : "${group.state}",
	 	 $2
	 	}
}
```
7. Add a funnel processor to the canvas and connect ReplaceText to it

#### Questions to Answer
1. What does a full RSVP Json object look like?
2. How many output files do you end up with?
3. How can you change the file name that Json is saved as from PutFile?
4. Why do you think we are splitting out the RSVP's by group?
5. Why are we using the Update Attribute processor to add a mime.type?
6. How can you cange the flow to get the member photo from the Json and download it.

------------------

# Lab 3

## Using Nifi template
   
In this lab, we will learn how to create, save, upload Nifi template and create Nifi flow using NiFi template.

### Goals:
   - Create Nifi template from exisitng flow
   - Save Template to xml file and upload xml template file to Nifi
   - Create new flow with existing template

1. Select everything inside Processor Group ```Lab 2```
	- Create a new template by click ```create template``` button
  	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-0.png)

2. Go to Template manager and download the newly created template to disk as xml file
	- In Template manager, click download button next to the newly created template to download it to local disk
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-1.png)
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-2.png)

3. Upload the template xml file from load disk to create another template
	- Change the template name inside the downloaded xml file so there will be no conflict when upload this template to Nifi
	```<name>Lab2TemplateNew</name>```
	- Click upload template button to upload the saved xml template file to Nifi
	- Create with a new name
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-3.png)

4. Create a new flow from existing templates
	- Create a new Processor Group ```Lab 3``` under ```Nifi Lab``` and double click to go inisde
	- Drag and drop template on canvas and select one of the template to create a new flow
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-4.png)
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab3-5.png)
	- Now you have a flow create from the template. There will be warning on Web socket procesor. This is caused by the controller service not enabled. Once you enable the controller service by clicking the flasj icon, everything works.

---------------------

# Lab 4

## Getting started with MiNiFi
   
In this lab, we will learn how to use Remote Process Group and use MiNiFi to send data to NiFi instance.

### Goals:
   - Understand how to communicate to remote Nifi instance
   - Prepare flow for MiNifi

## Setting up the Flow for NiFi
**NOTE:** Before start this lab, we need to enable Site-to-Site communication. Make the change via [Ambari UI](http://127.0.0.1:9080/) 

* Go to Nifi => Config => Advanced nifi-properties, and change the following values:
![Image]()

* Then restart NiFi via Ambari

Now we should be ready to create our flow. To do this do the following:

1. The first thing we are going to do is setup an Input Port. This is the port that MiNiFi will be sending data to. To do this drag the Input Port icon to the canvas and call it "From MiNiFi".

2. Now that the Input Port is configured we need to have somewhere for the data to go once we receive it. In this case we will keep it very simple and just log the attributes. To do this drag the Processor icon to the canvas and choose the LogAttribute processor.

3. Now that we have the input port and the processor to handle our data, we need to connect them. 

4. We are now ready to build the MiNiFi side of the flow. To do this do the following:
	- Add a GenerateFlowFile processor to the canvas (don't forget to configure the properties on it)
	- Add a Remote Processor Group to the canvas
	- Set the URL to ```http://sandbox-hdf:19090/nifi/```
 	- Connect the GenerateFlowFile to the Remote Process Group
	- Right click the Remote Process Group and Enable Transmission	

5. The next step is to generate the flow we need for MiNiFi. To do this do the following steps:
	- Create a template for MiNiFi 
	- Select the GenerateFlowFile and the NiFi Flow Remote Processor Group (these are the only things needed for MiMiFi)
	- Select the "Create Template" button from the toolbar
	- Choose a name for your template
	
7. Download the template to your local disk

8. Now SCP the template you downloaded to the ```/tmp``` directory on your VM. P
```
scp -P 12222 {path to MiNifi template} root@127.0.0.1:/tmp
root@127.0.0.1's password:hadoop
```

9. We are now ready to setup MiNiFi. However before doing that we need to convert the template to YAML format which MiNiFi uses. To do this we need to do the following:

	- Navigate to the minifi-toolkit directory (/usr/hdf/current/minifi/minifi-toolkit-0.2.0)
	- Transform the template that we downloaded using the following command:

      ```bin/config.sh transform <INPUT_TEMPLATE> <OUTPUT_FILE>```

      For example:

      ```bin/config.sh transform /temp/MiNiFi_Flow.xml config.yml```

10. Next copy the ```config.yml``` to the ```minifi-0.2.0/conf``` directory. That is the file that MiNiFi uses to generate the nifi.properties file and the flow.xml.gz for MiNiFi.

11. That is it, we are now ready to start MiNiFi. To start MiNiFi from a command prompt execute the following:

  ```
  cd /usr/hdf/current/minifi/minifi-0.2.0
  bin/minifi.sh start

  ```

You should be able to now go to your NiFi flow and see data coming in from MiNiFi.

------------------

# Lab 5

## Kafka Basics

In this lab we are going to explore creating, writing to and consuming Kafka topics. This will come in handy when we later integrate Kafka with NiFi.

### Goals:
   - Create kafka topic using console tool
   - send messages to kafka topic
   - receive messages from kafka topic

Before start the lab steps, make sure kafka service is started in [Ambari UI](http://127.0.0.1:9080/). If Kafka is not started, manually start the service from Ambari.

1. Creating a topic
  - Open an SSH connection to your VM.
  - Naviagte to the Kafka directory (````/usr/hdp/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

```
cd /usr/hdp/current/kafka-broker/
```

  - Create a topic using the kafka-topics.sh script
```
bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic first-topic
```

  - Ensure the topic was created
```
bin/kafka-topics.sh --list --zookeeper localhost:2181
```

2. Testing Producers and Consumers
  - Open a second terminal to your VM and navigate to the Kafka directory
  - In one shell window connect a consumer:
```
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic first-topic
```

  - In the second shell window connect a producer:
```
bin/kafka-console-producer.sh --broker-list sandbox-hdf:6667 --topic first-topic
```

  - Sending messages. Now that the producer is connected, we can type messages.
  - Type a message in the producer window

  - Messages should appear in the consumer window.

  - Close the consumer (ctrl-c) and reconnect using the default offset, of latest. You will now see only new messages typed in the producer window.

  - As you type messages in the producer window they should appear in the consumer window.

------------------

# Lab 6
  
## Integrating Kafka with NiFi

In this lab we will learn how to use Nifi to push data to kafka queue, as well as consume data from kafka queue. 

### Goals:
   - Send meetup JSON message to kafka queue using Nifi
   - Receive data from kafka queue using Nifi
   - Write data to local folder

1. Creating the Kafka topic
  - Open an SSH connection to your VM and naviagte to the Kafka directory
  - For our integration with NiFi create a Kafka topic called ```meetup-raw-rsvps```
```
bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic meetup-raw-rsvps
```

2. We are going to reuse the flow from Lab 2. Add a PublishKafka_0_10 processor to the canvas. Then connect the funnel to PublishKafka_0_10 processor.
	- Configuration PublishKafka_0_10 processor like the following
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab6-0.png)

3. Start the flow and using the Kafka tools verify the data is flowing all the way to Kafka.
```
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic meetup-raw-rsvps
```

4. Create a new Processor Group called ```Lab 6``` under ```Nifi Lab```.
	- Drag and drop ConsumeKafka_0_10 and config like the following
	- ![Image](https://github.com/pkuqiwang/nifi_lab/blob/master/lab6-1.png)
	- Drag adn drop PuFile processor and link the kafka processor to it, set ```Directory``` value to ```/tmp/nifilab```
	- start all processor and you will see the JSON data file get read from kafka queue and written to local folder 
	
------------------

