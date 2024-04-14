---
{"dg-publish":true,"permalink":"/hardware/building-an-anchor/","tags":["hardware","anchor"],"noteIcon":""}
---

> [!abstract] Guide to build an anchor
> This guide instructs you to create our [[Architecture#Hardware components\|Architecture#Hardware components]]' anchor. The anchor is the primary method for communication of the tag location to the system. The anchor and tag combination act as the base level units for the system.

## Components

As such the Anchor is made of three components electronically

- ESP32 Devkit V1
- DWM1000 UWB Module by Qorvo
- Custom PCB

## PCB Design and Build

The PCB is based of the following circuit.

![Anchor_circuit.png](/img/user/Attachments/hardware/Anchor_circuit.png)

*Fig 1: Anchor circuit diagram*

The gerber files for the PCB design can be found [here](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/PCB). The custom PCBs need to be manufactured prior to creating the anchor. The gerber files can be either sent to a manufacturer or manufactured locally using the given files in the zip folder.

![Anchor_PCB.png](/img/user/Attachments/hardware/Anchor_PCB.png)

*Fig 2: Anchor PCB*

As you can see there are pin holes for the ESP32 to be mounted. Ensure that the ESP32 is soldered onto the PCB in the correct orientation such that the Antenna of the ESP32 (An elongated flat black piece) is on the same side of the PCB marked antenna. The DWM1000 module is to be soldered onto the other side of the device such that its antenna faces away from the centre of the board. The orientations are crucial to the functioning of the anchor and therefore must be done with caution.

## Anchor Casing

Once the PCB is built, code can be uploaded into the ESP32 by following the instructions in this link. Once the anchor is setup, it can be placed into its casing. The casing currently for the prototype is 3D printed. The STEP models for the casing can be found [here](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/casing_model)

The STEP models can be exported into STL files and converted into gcode using the desired program for the 3D printer in use. The preferred material for the casing is PLA.

Once the Anchor models are printed they must be assembled as such.

![Anchor Assembly.png](/img/user/Attachments/hardware/Anchor%20Assembly.png)

Once the tag is completed if the ESP32 has not already been setup, you can upload the code from the following [link](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/anchor_code).

- Now move on to [[Hardware/Anchor Setup\|Anchor Setup]]
