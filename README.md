# Contents
- [Lab 1](#lab-1)
  - Build Environment
- [Lab 2](#lab-2)
  - Build a simple NiFi data flow
- [Lab 3](#lab-3) - MiNiFi
  - Enable Site2Site in NiFi
  - Designing MiNiFi Flow
  - Preparing the flow
  - Running MiNiFi
- [Lab 4](#lab-4) - Kafka Basics
  - Creating a topic
  - Producing data
  - Consuming data
- [Lab 5](#lab-5) - Integrating Kafka with NiFi
  - Creating the Kafka topic
  - Adding the Kafka producer processor
  - Verifying the data is flowing

## Goals:
  - Setup sandbox lab environment	
  - Consume Meetup RSVP stream
  - Extract the JSON elements we are interested in
  - Split the JSON into smaller fragments
  - Write the JSON to Kafka
  - Read the data from Kafka
  - Write data to local disk

We will run through a series of labs and step by step to achieve all of the above goals

------------------

# Lab 1

## Get HDF Sandbox

Download the HDF [Sandbox](https://hortonworks.com/downloads/#sandbox) from Hortonworks website.

All the following instrucitons are based on the [VirtualBox](https://www.virtualbox.org/wiki/Downloads) version of the Sandbox, if you use VMWare, you might need to make slight change to some of the settings.

## Use the Sandbox

Once the Sandbox is started in VirtualBox, the start page will show you the start page link like 

[http://127.0.0.1:18888](http://127.0.0.1:18888)

### To connect using Putty from Windows laptop

- Use putty to connect to your sandbox, password is "hadoop"
```
ssh root@127.0.0.1 -p 12222
```

### To connect from Linux/MacOSX laptop

- Use terminal to connect to your sandbox, password is "hadoop"
```
ssh root@127.0.0.1 -p 12222
```

### Reset Ambari password

- once you remote inside the sandbox, use following command to reset Ambari password
```
# Updates password
ambari-admin-password-reset
# If Ambari doesn't restart automatically, restart ambari service
ambari-agent restart
```

### Login to Ambari

- You could access Ambari UI at [http://127.0.0.1:18080](http://127.0.0.1:18080)

### NiFi Access

- You could access Nifi UI at [http://127.0.0.1:19090](http://127.0.0.1:19090)

-----------------------------

# Lab 2

## Goals:
   - Consume Meetup RSVP stream
   - Extract the JSON elements we are interested in
   - Split the JSON into smaller fragments
   - Use funnel as a temporary processor sink
   
## Consuming RSVP Data

To get started we need to consume the data from the Meetup RSVP stream, extract what we need, split the content and save it to a file:

Our final flow for this lab will look like the following:
![Image]()


  - Step 1: Add a ConnectWebSocket processor to the cavas
      - Configure the WebSocket Client Controller Service. The WebSocket URI for the meetups is: ```ws://stream.meetup.com/2/rsvps```
      - Set WebSocket Client ID to your favorite number.
      - ![Image]()
      
  - Step 2: Add an Update Attribute procesor
    - Configure it to have a custom property called ``` mime.type ``` with the value of ``` application/json ```
    - ![Image]()
    
  - Step 3. Add an EvaluateJsonPath processor and configure it as shown below:
  ![Image]()

    The properties to add are:
    ```
    event.name		$.event.event_name  
    event.url		$.event.event_url    
    group.city		$.group.group_city    
    group.state         $.group.group_state    
    group.country	$.group.group_country    
    group.name		$.group.group_name    
    venue.lat		$.venue.lat    
    venue.lon		$.venue.lon    
    venue.name		$.venue.venue_name
    ```
    
  - Step 4: Add a SplitJson processor and configure the JsonPath Expression to be ```$.group.group_topics ```
  - Step 5: Add a ReplaceText processor and configure the Search Value to be ```([{])([\S\s]+)([}])``` and the Replacement Value to be
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
  - Step 6: Add a PutFile processor to the canvas and configure it to write the data out to ```/tmp/rsvp-data```

##### Questions to Answer
1. What does a full RSVP Json object look like?
2. How many output files do you end up with?
3. How can you change the file name that Json is saved as from PutFile?
3. Why do you think we are splitting out the RSVP's by group?
4. Why are we using the Update Attribute processor to add a mime.type?
4. How can you cange the flow to get the member photo from the Json and download it.


------------------

# Lab 3

## Getting started with MiNiFi ##

In this lab, we will learn how configure MiNiFi to send data to NiFi:

* Setting up the Flow for NiFi
* Setting up the Flow for MiNiFi
* Preparing the flow for MiNiFi
* Configuring and starting MiNiFi
* Enjoying the data flow!


## Setting up the Flow for NiFi
**NOTE:** Before starting NiFi we need to enable Site-to-Site communication. To do that  we can either make the change via Ambari or edit the config by hand. In Ambari the below property values can be found at ````http://<EC2_NODE>:8080/#/main/services/NIFI/configs```` . To make the changes by hand do the following:

* Open /usr/hdf/current/nifi/conf/nifi.properties in your favorite editor
* Change:
  ````
			nifi.remote.input.host=
			nifi.remote.input.socket.port=
			nifi.remote.input.secure=true
  ````
  To
  ```
   		nifi.remote.input.socket.port=10000
			
  ```
* Restart NiFi via Ambari


Now we should be ready to create our flow. To do this do the following:

1.	The first thing we are going to do is setup an Input Port. This is the port that MiNiFi will be sending data to. To do this drag the Input Port icon to the canvas and call it "From MiNiFi".

2. Now that the Input Port is configured we need to have somewhere for the data to go once we receive it. In this case we will keep it very simple and just log the attributes. To do this drag the Processor icon to the canvas and choose the LogAttribute processor.

3.	Now that we have the input port and the processor to handle our data, we need to connect them. 

4.  We are now ready to build the MiNiFi side of the flow. To do this do the following:
	* Add a GenerateFlowFile processor to the canvas (don't forget to configure the properties on it)
	* Add a Remote Processor Group to the canvas

           For the URL copy and paste the URL for the NiFi UI from your browser
   * Connect the GenerateFlowFile to the Remote Process Group

5. The next step is to generate the flow we need for MiNiFi. To do this do the following steps:

   * Create a template for MiNiFi 
   * Select the GenerateFlowFile and the NiFi Flow Remote Processor Group (these are the only things needed for MiMiFi)
   * Select the "Create Template" button from the toolbar
   * Choose a name for your template
	

7. Now we need to download the template
8. Now SCP the template you downloaded to the ````/temp```` directory on your EC2 instance.
9.  We are now ready to setup MiNiFi. However before doing that we need to convert the template to YAML format which MiNiFi uses. To do this we need to do the following:

    * Navigate to the minifi-toolkit directory (/usr/hdf/current/minifi/minifi-toolkit-0.2.0)
    * Transform the template that we downloaded using the following command:

      ````bin/config.sh transform <INPUT_TEMPLATE> <OUTPUT_FILE>````

      For example:

      ````bin/config.sh transform /temp/MiNiFi_Flow.xml config.yml````

10. Next copy the ````config.yml```` to the ````minifi-0.2.0/conf```` directory. That is the file that MiNiFi uses to generate the nifi.properties file and the flow.xml.gz for MiNiFi.

11. That is it, we are now ready to start MiNiFi. To start MiNiFi from a command prompt execute the following:

  ```
  cd /usr/hdf/current/minifi/minifi-0.2.0
  bin/minifi.sh start

  ```

You should be able to now go to your NiFi flow and see data coming in from MiNiFi.

------------------

# Lab 4

## Kafka Basics
In this lab we are going to explore creating, writing to and consuming Kafka topics. This will come in handy when we later integrate Kafka with NiFi and Storm.

1. Creating a topic
  - Step 1: Open an SSH connection to your EC2 Node.
  - Step 2: Naviagte to the Kafka directory (````/usr/hdf/current/kafka-broker````), this is where Kafka is installed, we will use the utilities located in the bin directory.

    ````
    #cd /usr/hdf/current/kafka-broker/
    ````

  - Step 3: Create a topic using the kafka-topics.sh script
    ````
    bin/kafka-topics.sh --zookeeper localhost:2181 --create --partitions 1 --replication-factor 1 --topic first-topic

    ````

    NOTE: Based on how Kafka reports its metrics topics with a period ('.') or underscore ('_') may collide with metric names and should be avoided. If they cannot be avoided, then you should only use one of them.

  - Step 4:	Ensure the topic was created
    ````
    bin/kafka-topics.sh --list --zookeeper localhost:2181
    ````

2. Testing Producers and Consumers
  - Step 1: Open a second terminal to your EC2 node and navigate to the Kafka directory
  - In one shell window connect a consumer:
  ````
 bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic first-topic
````

    Note: using –from-beginning will tell the broker we want to consume from the first message in the topic. Otherwise it will be from the latest offset.

  - In the second shell window connect a producer:
````
bin/kafka-console-producer.sh --broker-list demo.hortonworks.com:6667 --topic first-topic
````


- Sending messages. Now that the producer is  connected  we can type messages.
  - Type a message in the producer window

- Messages should appear in the consumer window.

- Close the consumer (ctrl-c) and reconnect using the default offset, of latest. You will now see only new messages typed in the producer window.

- As you type messages in the producer window they should appear in the consumer window.

------------------

# Lab 5

## Integrating Kafka with NiFi
1.  Step 1: Creating the Kafka topic
  - For our integration with NiFi create a Kafka topic called ````meetup-raw-rsvps````


2. Step 2: Add a PublishKafka_0_10 processor to the canvas. It is up to you if you want to remove the PutFile or just add a routing for the success relationship to Kafka

3. Step 3: Start the flow and using the Kafka tools verify the data is flowing all the way to Kafka.


------------------

