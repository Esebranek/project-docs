# DIY Controller Area Network

![DIY CAN](https://raw.githubusercontent.com/Esebranek/project-docs/main/diy-can/header.jpeg)

> This project was originally published to Medium, but I have decided to re-host it on my own website for improved accessibility.

## Controller Area Networks (CAN)

A CAN is a bus communication system for connecting multiple microcontrollers. CANs are generally used in automotive solutions to connect a group of sensor devices, but other applications such as the exist. The strengths of a CAN lie in its prioritized messaging protocol, minimal hardware requirements, and absence of a host computer. Plug and play CAN hardware is available for consumer purchase, but in this post I will step through my experience building a network from basic consumer IoT components.

---

## My Network Configuration

Like any network, several factors should be considered in designing a CAN. For this project I opted to created a network with six nodes (the CAN module for Arduino came in packs of three). One node acts as a read only node, receiving data from the CAN and sending it out through serial. The other five will simply write data at random intervals. Normally these nodes would produce real data, but for the sake of simplicity I chose to send arbitrary data directly from the Arduino.

## Transmission

A CAN transmission line uses a single pair of wires. A high voltage wire will be driven up towards 5v, and a low voltage wire will be driven down towards 0. With these two wires the CAN module sends signals representing dominant 0 bits, or recessive 1 bits. Using dominant and recessive signals allows nodes to take priority or back off based on their ID.

## Bus Configuration

The easiest way to configure a CAN is with a single linear transmission line or linear bus. In my example this means placing two nodes on the end of a line and four branched off in the middle. Other applications may benefit from more complex configurations such as a star bus, or some combination of bus patterns. Although nodes are typically close in proximity, ISO 11898 cites capabilities of 1Mbs with a 25m line, and 50Kbs with a 1000m line. Since my network is on a breadboard, the node distances are short enough for high speed transmission.

## Termination

Since CANs use a transmission line to communicate, electrical termination needs to be addressed. This involves placing resistors on both ends of a network to prevent signal reflection and distortion. Also regulated with ISO 11898, the network should be terminated with 120Ω resistors. The CAN module I selected for this project has a built in 120Ω resistor that can be toggled by bridging a connection on the board.

## Message Structure

Each message sent within a CAN will contain an ID and data. The ID belongs to the sender node and will be either 11 or 29 bits long based on the CAN protocol used. A node with a larger ID will take priority when sending messages. The data sent in the message has a length of 8 bytes. There are a few more bits mixed into the message structure [described here](https://en.wikipedia.org/wiki/CAN_bus#Frames).

## Hardware

My configuration contains 6 nodes, with each node containing an Arduino and a CAN Bus module. I already had one Arduino UNO lying around so I commissioned it to be the read node since it can be easily distinguished and expanded upon in the future. For the other 5 nodes, I purchased a pack of Arduino Nano V3 boards. They use the ATMEGA328P and have more than enough digital pins, as well as the necessary VIN, 5V and GND. Plus I was able to get a pack of 10 for about $3.50 each. Each node also has a MCP2515 CAN Bus Arduino module which were about $3 a piece when I purchased them. In addition to the microcontrollers and CAN modules, I also picked up a 12V 2.5a DC Adapter to power the 5 sender nodes. All of these components were available from several sources on Amazon at the time.

| CAN MODULE                                                                                           | Arduino Nano                                                                                             | Arduino Uno                                                                                            |
| :--------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| ![CAN Module](https://raw.githubusercontent.com/Esebranek/project-docs/main/diy-can/can_module.jpeg) | ![Arduino Nano](https://raw.githubusercontent.com/Esebranek/project-docs/main/diy-can/arduino_nano.jpeg) | ![Arduino UNO](https://raw.githubusercontent.com/Esebranek/project-docs/main/diy-can/arduino_uno.jpeg) |

## Physical Set-up

Setting up the physical layer of my CAN was straightforward, but also a little tedious. The 5 Arduino Nanos required some initial work since all of the header pins had to be soldered into place on the board. From there, the CAN modules were connected to the Arduinos with 6 jumpers wires each. Then each Arduino Nano was connected to the 5V and ground rails of a breadboard. It should be mentioned that running 2.5a through a breadboard is bad practice since the boards generally aren’t rated for more than 1a–1.5a. However, I gave it a shot and didn’t have any issues which leads me to believe the nodes use less than their estimated 500ma. The Arduino UNO is connected to a PC and powered through the USB serial port. Connecting the nodes, creating the actual CAN, was the final piece of the puzzle. Each node’s high and low wire runs to a breadboard where columns for each node are bridged. My resulting network set-up is shown at the top of this post.

## Software Layer

Using one node as a receiver and five as senders meant I needed two scripts. In both, the CAN is initialized and then written to or read from. An Arduino library was needed to interface with the can module and can be retrieved from Github. I won’t go into anymore detail in hopes that the below gists are self documenting.

### Read Node Code

```
// READ

#include <SPI.h>
#include "mcp_can.h"

const int SPI_CS_PIN = 10;  // Define the chip select pin
MCP_CAN CAN(SPI_CS_PIN);  // Create can with the correct pin

void setup()
{
    Serial.begin(115200);  // Init serial with highest baudrate

    while (CAN_OK != CAN.begin(CAN_500KBPS))  // While initialization fails
    {
        Serial.println("Initialization failed. Retying...");
        delay(100);
    }
    Serial.println("Initialized.");
    Serial.println("Sender ID\tByte 0\tByte1\tByte2\tByte3\tByte4\tByte5\tByte6\tByte7");
}


void loop()
{
    unsigned char len = 0;
    unsigned char buf[8];

    if(CAN_MSGAVAIL == CAN.checkReceive()) // If we have a message available
    {
        CAN.readMsgBuf(&len, buf);  // Read the message into our buffer
        unsigned long senderID = CAN.getCanId();  // Get the sender nodes ID

        Serial.print(senderID, HEX);  // Print out sender
        Serial.print("\t\t");

        for(int i = 0; i<len; i++)  // Print out the data bytes
        {
            Serial.print(buf[i], HEX);
            Serial.print("\t");
        }
        Serial.println();
    }
}
```

### Write Node Code

```
//WRITE

#include <SPI.h>
#include <mcp_can.h>

const int SPI_CS_PIN = 10;  // Define chip select pin

MCP_CAN CAN(SPI_CS_PIN);  // Create

void setup()
{
    Serial.begin(115200);  // Initialize serial at highest baudrate

    while (CAN_OK != CAN.begin(CAN_500KBPS))  // While initialization fails retry
    {
        Serial.println("Initialization failed. Retrying...");
        delay(100);
    }
    Serial.println("Initialized.");
}

unsigned char msgBuf[8] = {0xA, 0xA, 0xA, 0xA, 0xA, 0xA, 0xA, 0xA};  // Use A-E for nodes 1-5

void loop()
{
  Serial.println("Sending data...");
  CAN.sendMsgBuf(0x01, 0, 8, msgBuf);  // Update nodes ID (0x01 - 0x05)
  delay(random(250,1000));  // Wait some time before sending more
}
```

## Powering Up

Next the code was uploaded to each Arduino and the network was powered on. Using the Arduino IDE serial monitor I was able to capture the serial output of the read node. The random send interval allowed data from all 5 write nodes to get through without being blocked by a higher priority node. Success!

![Results](https://raw.githubusercontent.com/Esebranek/project-docs/main/diy-can/results.png)

## Next Steps

Going into this project I had no knowledge of CANs, but I was comfortable working with was DIY electronics. After reading up on the technology, I did a quick search for Arduino CAN set-ups and found a few resources. None of the examples appeared comprehensive, but I was able to piece together information to get a functional DIY CAN of my own. Going forward I’d like to take this simple CAN set-up and expand both ends of the network to do something more interesting. Perhaps that means generating real data with a handful of sensors, or viewing the data in some way other than a serial monitor. Regardless of the next steps, building a DIY CAN was an informative and enjoyable experience.
