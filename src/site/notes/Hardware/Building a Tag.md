---
{"dg-publish":true,"permalink":"/hardware/building-a-tag/","tags":["hardware","tag"],"noteIcon":""}
---

> [!abstract] Tag prototype creation instructions
> This guide teaches you how to build a tag. The tag is attached to the object of interest and acts as the tracker attached to it. It constantly sends messages back and forth from anchor and tag using TWR to measure the distance between the anchor and tag. The anchor and tag combination act as the base level units for the system.

## Components

As such the Tag is made of these components electronically

- ESP32 Devkit 1
- DWM1000 UWB Module
- Accelerometer MPU6050
- TP4056 Battery Charging Module
- 3.7V 1500mAh Battery (LiPO)
- Pressure Sensor (Optional)
- DF9-40 Pressure Senstivity Resistor (Optional)
- 200k Ohm Resistor (Optional, used parallel to DF9-40)
- Custom PCB

The PCB is based of the following circuit.

![Tag_circuit.png](/img/user/Attachments/hardware/Tag_circuit.png)

*Fig 1: Tag circuit diagram*

The two major components are the ESP32 and the DWM1000. The gerber files for the PCB design can be found below. The custom PCBs need to be manufactured prior to creating the Tag. The gerber files can be either sent to a manufacturer or manufactured locally using the given files in the zip folder. They can be found [here](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/PCB).

![Tag_PCB.png](/img/user/Attachments/hardware/Tag_PCB.png)

*Fig 2: Tag PCB*

As you can see there are pin holes for the ESP32 to be mounted. The tag also includes components like the accelerometer and the pressure for the detachment mechanism to add to the additional physical security of the Tag. The accelerometer also enables the tag to go into sleep mode until it is disturbed by external forces, allowing the battery to last longer.

Once the PCB is manufactured the ESP32 must be soldered onto the PCB in the correct orientation such that the Antenna of the ESP32 (An elongated flat black piece) is on the same side of the PCB marked antenna. The DWM1000 module is to be soldered onto the same side of the device such that its antenna faces away from the centre of the board. The orientations are crucial to the functioning of the anchor and therefore must be done with caution. The accelerometer should be soldered into the right side of the PCB. Similarly the two output pins of the battery charging module are located on the right side as well. The pressure sensor pins can be soldered into the top beside the UWB module.

>[!note]
>If the tag is being built without the detachment mechanism then the pressure sensor and the resistor do not need to be soldered on.

## Tag Component Assembly:

1. Solder on DWM1000 UWB module on custom PCB with castellated mounting holes.
2. Solder on ESP32-WROOM-32 microcontroller
	1. **IMPORTANT!** As we are using a 3.7V LiPo battery to power the tag, ensure that the ESP32 is capable of receiving voltages from 3V to 4.2V. This is dependent on the on-board LDO.
	2. If the ESP32 is unable to receive a varied voltage, consider implementing an external LDO to regulate the voltage, or redesign a custom ESP32 development board for this use case (Ideal).
3. Solder on MPU6050 accelerometer.
4. Snip off excess pin lengths
	1. This is to ensure that the pins would not puncture the LiPo battery upon mounting into the casing
5. Solder the 3.7V LiPo battery onto the B+ and B- terminals of the TP4056 battery charging module respectively
	1. Check the connection by briefly charging the battery via the USB-C port on the battery charging module.
	2. **IMPORTANT!** If the charging does not seem to work, try using a USB-A to USB-C cable. USB-C cables have a function where it will stop power supply upon no/incorrect detection resistors, and the Chinese OEM TP4056 boards we bought did NOT have it, thus preventing the cable from supplying power. USB-A to USB-C cables however do not have this requirement and will supply power as per normal.
6. Solder 2 cables from the Out+ and Out- of the TP4056 board to the pads on the back of the custom PCB.
	1. **IMPORTANT!** There was no switch integrated in this device as there was no need to turn it off. Be careful to not let the positive and negative terminals touch when soldering, as it might cause sparks and/or an electrical fire
7. *(OPTIONAL)* If assembling for tags with a detachment functionality, solder the 200k Ohm onto the custom PCB, and the force sensor on the back of the custom PCB
8. The circuitry components for the tag is completed

## Tag Casing without Detachment

Once the PCB is built, code can be uploaded into the ESP32 by following the instructions in this link. Once the tag is setup, it can be placed into its casing. The casing currently for the prototype is 3D printed. The STEP models for the casing can be found [here](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/casing_model)

For building tags without the detachment mechanism use the models in `Tag Casing`.

The STEP models can be exported into STL files and converted into gcode using the desired program for the 3D printer in use. The preferred material for the casing is PLA.

Once the Tag casing models are printed they must be assembled as such. There are mainly two components to the casing the outer and the inner covering. They can be assembled using the inbuilt snap fit mechanism which requires you to push the two parts of the casing together as shown in the figure below.

![Tag_assembly_without_Detach.png](/img/user/Attachments/hardware/Tag_assembly_without_Detach.png)

*Fig 3: Tag PCB assembly without Detachment*

## Tag Casing with Detachment

The `Tag Casing (Latch)` contains the extra detachment mechanism. This also needs to be printed using PLA. For the detachment mechanism to work it requires an extra silicon bead that is inserted into the main casing which presses down on the pressure sensor. The silicon bead must be inserted into the inner layer as shown in the figure below.

![Detachment_bead_install.png](/img/user/Attachments/hardware/Detachment_bead_install.png)

*Fig 4: Silicon Bead Installation*

Once the bead is installed the latch must be loaded into the back casing to make sure that the pressure sensor can have force exerted when it is attached to an object. The latch can be added in the way shown in Fig 5.

![Latch_assembly.png](/img/user/Attachments/hardware/Latch_assembly.png)

*Fig 5: Latch Assembly*

Once the latch and bead have been inserted they can be assembled using the inbuilt snap fit mechanism which requires you to push the two parts of the casing together as shown in the figure below. Please pay attention to the placement of the pressure sensor and the battery for this configuration of the Tag.

![Tag latch.png](/img/user/Attachments/hardware/Tag%20latch.png)

*Fig 6: Top case with tag*

Once you have this you can assemble it with the rest.

![Tag_assembly_with_detach.png](/img/user/Attachments/hardware/Tag_assembly_with_detach.png)

*Fig 7: Tag PCB assembly with Detachment*

Once the tag is completed if the ESP32 has not already been setup, you can upload the code from the following [link](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/tag_code).

- Now you can go [[Hardware/Tag Setup\|Tag Setup]]