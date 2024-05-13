---
dg-publish: true
tags:
  - hardware
  - tag
---
> [!abstract] Guide to setup a built tag
> This guide assumes that [[Hardware/Building a Tag\|Building a Tag]] has been complete. This illustrates how to setup a tag.

## Setup

Once the components are soldered together, the Tag is ready for programming. For the purposes of programming the ESP32, we use the Arduino IDE.

- Navigate to the Arduino Folder in the Documents Folder of your PC or Laptop that you are programming the ESP32 on.
- Open the libraries folder and copy over the Arduino libraries contained in the package.

Ensure that the following libraries are present within the device.

- DWM1000Ng
- Adafruit MPU6050
- Adafruit Libraries

## Basic Functionality

Connect the ESP32 via the serial port to your PC or laptop using the USB port. Make sure the right device is selected on the dropdown menu for the different Serial ports.

>[!note]
> Before you begin coding the ESP32 make sure you
>
> - Set the Tag Unique ID
> - Open Arduino IDE open a new file and save it in a folder with the same name.

### Necessary Libraries

 We will first add in all the necessary libraries for the code to function

```C
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <Wire.h>
#include <DW1000Ng.hpp>
#include <DW1000NgUtils.hpp>
#include <DW1000NgTime.hpp>
#include <DW1000NgConstants.hpp>
#include <DW1000NgRanging.hpp>
#include <DW1000NgRTLS.hpp>
```

The DW1000Ng libraries are used exclusively for UWB communication from Anchor to Tag. We use the Adafruit Libraries to set up the accelerometer for the Tag. This accelerometer controls the sleep mode and establishes when the tag is in motion.

### Structures, Configurations and Declarations

There are specific configurations that need to be instantiated to enable efficient working of the DWM1000 module with the existing code.

```C
Adafruit_MPU6050 mpu;

Adafruit_MPU6050 mpu;
#if defined(ESP8266)
const uint8_t PIN_SS = 5;
#else
const uint8_t PIN_SS = 5; // spi select pin
const uint8_t PIN_RST = 35;
#endif

static byte SEQ_NUMBER = 0;

#define FORCE_SENSOR_PIN 32 
```

These are the SPI pin values for the ESP32 DevKit V1 that is unique to the board. If there are modifications to the board and the PCB these values need to be updated. The Force sensor pin is initialised as 32 which is the pin that reads from the pressure sensor.

```C
volatile uint32_t blink_rate = 200;
bool detach;

device_configuration_t DEFAULT_CONFIG = {
    false,
    true,
    true,
    true,
    false,
    SFDMode::STANDARD_SFD,
    Channel::CHANNEL_5,
    DataRate::RATE_850KBPS,
    PulseFrequency::FREQ_16MHZ,
    PreambleLength::LEN_256,
    PreambleCode::CODE_3
};

frame_filtering_configuration_t TAG_FRAME_FILTER_CONFIG = {
    false,
    false,
    true,
    false,
    false,
    false,
    false,
    false
};

sleep_configuration_t SLEEP_CONFIG = {
    false,  // onWakeUpRunADC   reg 0x2C:00
    false,  // onWakeUpReceive
    false,  // onWakeUpLoadEUI
    true,   // onWakeUpLoadL64Param
    true,   // preserveSleep
    true,   // enableSLP    reg 0x2C:06
    false,  // enableWakePIN
    true    // enableWakeSPI
};
```

These following lines of code initialise different configurations that will be used in the later parts of the code especially during the ranging process and the setup of the devices.

### Tag ID

The Tag Unique ID needs to be initialised as well. Below is an example of the EUI. It is a 16 character code where the values can be from '0 to 9' and 'A to F'. Only the first 12 digits should be modified as the last 4 digits depict the state of the Tag.

Here the EUI last 4 digits explain the states of the Tag, `EUI_new` to establish movement of the tag and `EUI_detach` for when the tag is detached.

```C
// Extended Unique Identifier register. 64-bit device identifier. Register file: 0x01
char EUI[] = "00:03:11:DD:EE:FF:00:00";
char EUI_new[] = "00:03:FF:DD:EE:FF:00:00";
char EUI_detach[] = "00:03:FF:DD:EE:FF:00:01";
```

Currently the last 4 digits '00:00' stand for stationary and alive. Signals sent with this state will not cause any alerts unless they are in the wrong location.

> [!note]
>The four states of the Tag is as follows:
>
> - `00:00`: Tag is `stationary` and battery is `high`
> - `00:01`: Tag is `detached`and battery is `high`
> - `01:00`: Tag is `stationary` and battery is `low`
> - `01:01`: Tag is `detached` and battery is `low`
>
> The EUIs must be initialised for each tag with the first 12 unique digits for each tag.

Our current prototype does not allow for testing battery power remaining. Thus it only contains EUI states for detachment.

### Setup and Loop

The setup and the loop are the most important sections of the code as they are what actually run the Tag. The setup loop which will run once before the loop begins executing. Most of what occurs in the setup loop is the initialisation of configurations

```C
void setup() {
    // DEBUG monitoring
    Serial.begin(115200);
    delay(1000);
    Serial.println(F("### DW1000Ng-arduino-ranging-tag ###"));

    // initialize the driver
    #if defined(ESP8266)
    DW1000Ng::initializeNoInterrupt(PIN_SS);
    #else
    DW1000Ng::initializeNoInterrupt(PIN_SS, PIN_RST);
    #endif
    Serial.println("DW1000Ng initialized ...");

    // general configuration
    DW1000Ng::applyConfiguration(DEFAULT_CONFIG);
    DW1000Ng::enableFrameFiltering(TAG_FRAME_FILTER_CONFIG);
    DW1000Ng::setEUI(EUI);
    DW1000Ng::setNetworkId(RTLS_APP_ID);
    DW1000Ng::setAntennaDelay(16436);
    DW1000Ng::applySleepConfiguration(SLEEP_CONFIG);
    DW1000Ng::setPreambleDetectionTimeout(15);
    DW1000Ng::setSfdDetectionTimeout(273);
    DW1000Ng::setReceiveFrameWaitTimeoutPeriod(2000);
    Serial.println(F("Committed configuration ..."));

    // DEBUG chip info and registers pretty printed
    char msg[128];
    DW1000Ng::getPrintableDeviceIdentifier(msg);
    Serial.print("Device ID: "); Serial.println(msg);
    DW1000Ng::getPrintableExtendedUniqueIdentifier(msg);
    Serial.print("Unique ID: "); Serial.println(msg);
    DW1000Ng::getPrintableNetworkIdAndShortAddress(msg);
    Serial.print("Network ID & Device Address: "); Serial.println(msg);
    DW1000Ng::getPrintableDeviceMode(msg);
    Serial.print("Device mode: "); Serial.println(msg); 

    // Initialise accelerometer
    if (!mpu.begin()) {
	    Serial.println("Failed to find MPU6050 chip");
	    while (1) {
	      delay(10);
	    }
    }
    Serial.println("MPU6050 Found!");

    //setupt motion detection
    mpu.setHighPassFilter(MPU6050_HIGHPASS_0_63_HZ);
    mpu.setMotionDetectionThreshold(1);
    mpu.setMotionDetectionDuration(20);
    mpu.setInterruptPinLatch(true);  // Keep it latched.  Will turn off when reinitialized.
    mpu.setInterruptPinPolarity(true);
    mpu.setMotionInterrupt(true);   
}
```

Here we setup and initialise both the UWB module and accelerometer for the tags functioning purposes. These parameters control the function of both these components and how and when they are used.

```C
void loop() {

  // Check if detachment occurs
  if (pressure_sense()){
    DW1000Ng::deepSleep();
    delay(blink_rate);
    DW1000Ng::spiWakeup();
    DW1000Ng::setEUI(EUI_detach);
    DW1000Ng::setTransmitData(EUI_detach);

    RangeInfrastructureResult res = DW1000NgRTLS::tagTwrLocalize(1500);
    
    if(res.success)
		blink_rate = res.new_blink_rate;
    Serial.println("SentTWR");
  }

  //Motion detection : Starts transmitting
  else{
  if(mpu.getMotionInterruptStatus()) {
    /* Get new sensor events with the readings */
    sensors_event_t a, g, temp;
    mpu.getEvent(&a, &g, &temp); 
    DW1000Ng::deepSleep();
    delay(blink_rate);
    DW1000Ng::spiWakeup();
    DW1000Ng::setEUI(EUI_new);
    DW1000Ng::setTransmitData(EUI_new);

    RangeInfrastructureResult res = DW1000NgRTLS::tagTwrLocalize(1500);
    
    if(res.success)
	blink_rate = res.new_blink_rate;

    /* Print out the values for debugging */
    Serial.print("AccelX:");
    Serial.print(a.acceleration.x);
    Serial.print("sENT DATA");
    Serial.println(EUI);
  }

  // Send heartbeat signals for aliveness checks
  else{
  static unsigned long lastCheckTime = 0;
  unsigned long currentTime = millis();
  const unsigned long retryInterval = 10000; // retry every 10s
  static unsigned long index = 0;
  const unsigned long sendSignal = 10000;

  // Only attempt to send every 10s
  if (currentTime - lastCheckTime >= retryInterval) {
    lastCheckTime = currentTime;
    DW1000Ng::deepSleep();
    delay(blink_rate);
    DW1000Ng::spiWakeup();
    DW1000Ng::setEUI(EUI_new);
    DW1000Ng::setTransmitData(EUI_new);

    RangeInfrastructureResult res = DW1000NgRTLS::tagTwrLocalize(1500);
    
    if(res.success)
    blink_rate = res.new_blink_rate;
    Serial.println("Alive");}
  }
  } 
}
```

In the main loop we initially check to see if the tag is detached. If it is it immediately begins to constantly transmit. If not we check if any motion is detected by the Tag. If the accelerometer and gyroscope are detecting changes in acceleration or rotation then the Tag starts to continually transmit. If no motion is detected than the tag only sends out a series of signals over a period of 10 seconds. This is its heartbeat or aliveness message.

## Conclusion

Once the Tags are completely set up and their unique ID is uploaded and Initialised, the tag code can be uploaded. However in order to test the tag the anchor must also be built and formatted. The instructions for building the anchor can be found [[Hardware/Building an Anchor\|here]] and the instructions for formatting the anchor can be found [[Hardware/Anchor Setup\|here]].

## References

Sewio. (2024, January 18). Analyzing decawave two way ranging (TWR). Sewio RTLS. https://www.sewio.net/uwb-sniffer/analyzing-decawave-two-way-ranging-twr/

F-Army. (n.d.). F-Army/Arduino-DW1000-NG: Arduino driver and library to use decawave’s DW1000 IC and relative modules. GitHub. https://github.com/F-Army/arduino-dw1000-ng
