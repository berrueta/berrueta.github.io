---
layout: post
title:  "LEGO trains distributed control system using Raspberry Pi, AWS and Trello"
date:   2023-08-26 15:49:29 +1200
category: software
tags: lego aws raspberry-pi lambda python trains power-up sensor track maker
---
(This is a re-publish of the documentation of a project that I did in 2019.
Please go to [Bitbucket](https://bitbucket.org/berrueta/trains) to find the source code.)

This document describes a distributed control system for LEGO trains using Raspberry Pi, AWS and Trello.

To see a demo, please check this video:

{% include youtube.html id="_Gh2BcDhJGw" %}

I created this system as a hobby and to teach myself some electronics, AWS IoT and serverless. Needless to say,
this is not the most practical or precise way to control vehicles. For example, the latency of the system is sometimes
measured in seconds, which leads to trains occasionally overshooting their marks, entering the wrong track or
crashing against each other. If you are serious about real-time traffic control, you probably should look
somewhere else.  

## Hardware

### Components

* LEGO PowerUp trains (Bluetooth enabled). Note that previous LEGO PowerFunctions trains did not use Bluetooth.
* LEGO round magnets (part 73092). At least as many magnets as train engines.
* LEGO-compatible automated track switches using servo motors. I used the [ones sold by 4DBrix](https://www.4dbrix.com/products/train/track-switch-motor/).
* KY-003 Hall effect sensors. As many as blocks in the track. 
* MCP23017 16-bit I/O port.
* PCA9685 12-bit PWM/Servo driver.
* Raspberry Pi. I used Raspberry Pi Zero, but other models should work too.
* Breadboard, breadboard power supply with dual 3.3V and 5V outputs, Raspberry Pi T-Cobbler and 40-pin cable.
* Breadboard jumpers and servo lead extension cables.
* (Optional) A LED, a push button and a 220 Î© resistor.

### Train setup

The only change required to the trains is the addition of a LEGO magnet at the underside of the engine. You may need to
pad it with a 2x2 flat brick to the magnet sits lower, almost at the level of the track. Only one magnet is needed
per train. Additional magnets to other cars for redundancy, but they are not necessary.

![Picture of the underside of the LEGO train engine, showing the location of the magnet](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/IMG_6903.jpeg)

The purpose of this magnet is to excite the Hall effect sensor under the track. If the sensor is not triggered
when the train passes over it, please check the polarity of the magnet (you may need to reverse it) and
reduce the distance between the magnet and the sensor by adding more padding.

### Track setup

The track must form a circuit. At the moment the control system only supports one-way routing, so make sure the
track forms a loop and does not reverse the direction.

For traffic management and routing purposes, the track must be conceptually partitioned into multiple "blocks".
To avoid collisions, the control system ensures that only one train occupies a block at any given time. The
blocks must be longer than the trains, with generous buffer to account of the imprecision of the control system.

The following diagram represents a possible track that consists of 4 blocks, 4 sensors and 1 automated switch.
In this circuit, trains always move in clockwise direction. 

![Diagram of a possible track](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/track.png)

Blocks are delimited by the Hall effect sensors that detect when the magnet passes over them. Due to the
imprecision of the control system, trains will not stop directly over the magnet, but a bit later. Take that
into account, e.g. leave a buffer space after the sensor if there is a track switch following the sensor.
In order to install the sensor under the track, you may need to elevate the track slightly and add a few pieces to
secure the sensor. Note that the active part of the sensor must be located right in the middle of the track,
aligned with the magnet under the train.

![Picture of the Hall effect sensor installed under the track](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/IMG_6897.jpeg)

Automated track switches use a servo to replace the manual lever that comes with the official LEGO track switch.
They must be used where the track forks. They are not strictly required when the track merges again, as the
train will be able to merge even if the switch rails are not aligned with the train direction.

A switch should be preceded by a sensor, to allow the trains to stop just before the switch in case its destination
block is busy. 

![Picture of an automated track switch](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/IMG_6901.jpeg)

### Control system setup

The control system is based on a Raspberry Pi. It interacts with the LEGO PowerUp engines using Bluetooth, and
with the AWS cloud using WiFi. In order to support up to 16 switches and 16 sensors, it communicates via the I2C
bus with the MCP23017 and PCA9685 chips (if necessary, more of these can be added to the bus).

![Control system sketch diagram](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/sketch.png)

The KY-003 Hall effect sensors are triggered when the train with the magnet passes over them. They send a signal
to one of the pins of the MCP23017, which triggers an interruption. The control program running in the Raspberry Pi
reads the signal, clears the interrupt flag and sends a MQTT message to AWS IoT. The diagram shows only one
of these sensors, but many of those are necessary and can be connected to the pins of the MCP23017. The optional
LED indicates when a new reading is available from a sensor, for debugging purposes.

The PCA9685 chip allows the Raspberry Pi to control multiple servo motors, in this case the switches. Only one
switch is depicted in the diagram, but more of them can be connected as necessary.

Note that the MCP23017 and PCA9685 must use different addresses in the I2C bus.

An optional push button provides a convenient way to reset the state of the routing system. 

## Software

### Architecture

This is a distributed control system that runs partially in the Raspberry Pi and partially in the AWS cloud.
All the control logic and state is located in the AWS cloud, and the Raspberry Pi is merely a sophisticated
interface between the physical devices (sensors, switches and of course the trains) and the cloud. The
communication between the Raspberry Pi and AWS happens via the MQTT protocol and AWS IoT Core. Four MQTT topics
are used to exchange four types of messages:

* Train speed commands: sent from the cloud to adjust the speed of a train. It can be "stop", "advance at low speed"
or "advance at high speed".
* Switch command: send from the cloud to adjust the position of a switch. It can be either "straight" or "turn".
* Detector message: send to the cloud to indicate that a sensor has been triggered (the train has reached a mark).
* Reset message: sent to the cloud to reset the state of the routing algorithm.

![Deployment diagram of the distributed control system](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/diagram.png)

Due to limitations of the libraries used, the software that runs in the Raspberry Pi has been split in two
Python programs, both of them in the `embedded` directory:

* `control.py`: receives MQTT messages from AWS and interacts with the switches (via the PCA9685 chip) and the
trains (via Bluetooth).

* `sensors.py`: monitors the sensors via the MCP23017 chip and sends MQTT messages to AWS. Additionally, it also
sends messages when the reset button is pushed.

In addition, the `lambda` directory contains an AWS SAM application that uses Lambda and Dynamodb to control
the system. It keeps track of the state of the system and makes routing and start/stop decisions.

### Traffic control algorithm

The traffic control algorithm is a serverless application that runs in the AWS cloud. It is composed of a number of
lambda functions and Dynamodb tables.

The algorithm models the track as a directed graph, where each block is represented by a vertex.
Each vertex is labelled with the identifier of a sensor.
A directed edge between two vertices indicates that the blocks form a continuous track. An edge can also be labelled
with a pair that indicates the position of a switch that connects the two blocks.

The following snippet defines a simple circuit with 4 blocks, 4 sensors and 1 switch. This is the same circuit depicted
in the diagram above, and also the same one that appears in the video demo:

```python
    graph = nx.DiGraph()

    graph.add_node("block_1", detector_id="detector_8")
    graph.add_node("block_2", detector_id="detector_9")
    graph.add_node("block_3", detector_id="detector_10")
    graph.add_node("block_4", detector_id="detector_11")

    graph.add_edges_from([
        ("block_1", "block_3"),
        ("block_2", "block_3"),
        ("block_3", "block_4"),
        ("block_4", "block_1", {'switch_id': 'switch_1', 'switch_position': 'straight'}),
        ("block_4", "block_2", {'switch_id': 'switch_1', 'switch_position': 'turn'})
    ])
```

A visual representation of the same graph:

![A graph that represents a simple circuit](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/graph.png)

Train routing becomes a problem of path finding between the vertex that represents the current location of the train
and the vertex that represents its destination. Due to the construction of the graph, the path is guaranteed to exist.
However, since there may be more than one train, it is necessary to prevent collisions.

The collision prevention algorithm is based on the idea of "warrants". A warrant is a sequence of vertices that is
reserved for exclusive use of a particular train. Until the train releases the warrant, no other train can enter any
of the vertices (blocks) included in the warrant. Let's say train A wants to go from block 1 to block 2. Train B is
stationed in block 2 at the moment. A path will be calculated for train A, that includes block 1 (origin), followed by block 3,
followed by block 4, followed by block 2 after moving the switch 1 to the "turn" position. However, train A cannot
be given a warrant over the full path, because train B is occupying block 2. Therefore, train A will be given a
warrant over block 1, block 3 and block 4, excluding block 2, and can start moving immediately. It will continue moving
until it reaches the end of the warrant (block 4), and it will stop there until block 2 becomes available and it can
continue its journey. As the train moves along the track, it will trigger the sensors. For example, when the train
passes over sensor 10, which is associated with block 3, the system will know that the train has progressed as far as
block 3, and therefore block 1 can be released from the warrant. Similarly, when the train passes over sensor 11, which
is associated with block 3, the block 3 is released from the warrant.

In order to model the state of the system, the algorithm maintains two Dynamodb tables. The `trains` table
represents the state of the trains:

| train_id | bluetooth_id | speed | destination | warrant |
| ---- | ---- | ---- | ---- | ---- |
| Train A | aabbcc | 30 | block_2 | block_1, block_3, block_4 |
| Train B | ddeeff | 0 | block_3 | block_2 |

And the `switches` table represents the state of the track switches:

| switch_id | position |
| ---- | ---- |
| Switch 1 | turn |

The traffic control system follows a reactive MVC-like pattern, where the Dynamodb tables act as the model. The routing
algorithm, implemented as an AWS Lambda function, does not interact with the sensors, switches or trains,
neither directly nor via MQTT.
Instead, it updates the Dynamodb tables. For example, the result of running the routing function will be some
updates to the train `speed` and `warrant` fields, and the `position` of the switches. Other Lambda functions are
triggered by the changes in the tables (using Dynamodb Streams) and send MQTT messages. The routing function itself
is also triggered in reaction to changes in the tables, for example a change in the `destination` field of a train,
or MQTT events from the sensors. In other words, the control system is not running in a continuous loop, but instead
it is composed of a number of reactive Lambda functions.

### Prerequisites

* Python 3.7.
* Install AWS SDK and AWS SAM: ```pip3.7 install --user aws-sam-cli awscli```
* An AWS account.
* An Atlassian account (Trello).

### Running the SAM application locally

1. Run the application: ```sam local start-api```


### Deploying the SAM application

1. Define an S3 bucket name: ```export BUCKET_NAME=berrueta-trains-deployment```

1. Create a S3 bucket (only necessary the first time): ```aws s3 mb s3://${BUCKET_NAME}```
  
1. Package the app: ```sam package --output-template-file packaged.yml --s3-bucket ${BUCKET_NAME}```

1. Deploy it: ```aws cloudformation deploy --template-file packaged.yml --stack-name TrainStack1```

1. Optionally, you can use ```aws cloudformation describe-stack-events --stack-name TrainStack1``` to see the CloudWatch activity.


### Registering the thing with AWS IoT Core

1. Go to the AWS IoT Core console and add a two new 'things', called 'sensors' and 'control'. Generate and download the certificates
to `embedded/certs`.

2. In the AWS IoT Core console, attach the `AccessMQTT` policy to the new certificates. This policy should look like:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:*",
      "Resource": "*"
    }
  ]
}
```


### Running the embedded applications

1. Copy the directory `embedded` to the Raspberry Pi.

1. Put the AWS certificates into a new `embedded/certs` subdirectory.

1. SSH into the Raspberry Pi and go to the `embedded` directory

1. Install the dependencies: ```pip3.7 install -r requirements.txt```

1. You may need to update the `thing_name` and `host` variables in `control.py` and `sensors.py`.

1. Run the `control` application that controls the switches and the train engines: ```sudo python3.7 control.py``` (only works in a Raspberry Pi, remember to copy the certificates)

1. In another SSH terminal, run the application that monitors the sensors: ```sudo python3.7 sensors.py```


### Trello control dashboard

The UI to give orders (i.e., set the destination) to the trains has been built as a Trello board. In this board
there is a list for each of the blocks in the track. Trains are represented as cards. Moving a card to a different
list sets the desired destination for that train. 

![Screenshot of a Trello board with trains located under lists that represents blocks](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/trello-board.png)

The train card description includes the Bluetooth ID of the train, so it can be matched against the model in the
`trains` table in Dynamodb. Other attributes of the card, like the title, the pictures and the labels are purely
for informative and decorative purposes. 

![Screenshot of a Trello card showing the details of a train](https://bytebucket.org/berrueta/trains/raw/c1a710505efbfc9d1d1e195826a49b7df33fb554/docs/trello-card.png)

Changes in the Trello board are pushed to the control system by registering a Trello webhook that points to one
of the Lambda functions that is exposed via AWS API Gateway. This registration must be done manually, please check
the [Trello documentation](https://developer.atlassian.com/cloud/trello/guides/rest-api/webhooks/) for details.

Note that the board does not represent the current location of the trains. It represents the intended destination
for each train. The board does not update as the trains move (although that would be a nice future improvement!).

Other human interfaces could be also built, for example using Alexa to give orders to the trains.
