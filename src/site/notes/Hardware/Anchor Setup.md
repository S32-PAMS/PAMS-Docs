---
dg-publish: true
tags:
  - anchor
  - hardware
---
> [!abstract] Guide to setup anchors
> This guide assumes you have completed [[Hardware/Building an Anchor\|Building an Anchor]]. The anchor is the primary method for communication of the tag location to the system. The anchor and tag combination act as the base level units for the system.

> [!important]
> You must have completed [[Server/Server initial setup#Protobuf compilation\|Server initial setup#Protobuf compilation]]!

## Setup

The two major components are the ESP32 and the DWM1000, on the PCB.

Once the three components are soldered together, the Anchor is ready for programming. For the purposes of programming the ESP32, we use the Arduino IDE.
- Navigate to the Arduino Folder in the Documents Folder of your PC or Laptop that you are programming the ESP32 on.
- Open the libraries folder and copy over the Arduino libraries contained in the package.

Ensure that the following libraries are present within the device. They need to be downloaded into the libraries folder in the location in which Arduino is stored in the device. Usually this can be found in `Documents/Arduino/Libraries` filepath on your device. The dowloadable files can be found [here](https://github.com/S32-PAMS/PAMS-Hardware/tree/3bb20bb0ad1d8bf1a5af60a0f52359bf73bfcf76/arduino_libraries)

- DWM1000Ng
- Messages
- NanoPb
- PubSubClient

## Basic Functionality

Connect the ESP32 via the serial port to your PC or laptop using the USB port. Make sure the right device is selected on the dropdown menu for the different Serial ports.

>[!note]
> Before you begin coding the ESP32 make sure you have the following details:
>
> - Wifi Name
> - Wifi Password
> - MQTT Broker IP Address
> - MQTT Broker Username and Password (If any)
> - Anchor Unique ID - for the new Anchor

In the Arduino IDE open a new file and save it in a folder with the same name.

### Necessary Libraries

 We will first add in all the necessary libraries for the code to function

```C
#include <WiFi.h>
#include <PubSubClient.h>
#include "pb_common.h"
#include "pb.h"
#include "pb_encode.h"
#include "message.pb.h"
#include "time.h"
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <string>

#include <DW1000Ng.hpp>
#include <DW1000NgUtils.hpp>
#include <DW1000NgRanging.hpp>
#include <DW1000NgRTLS.hpp> 
```

The Wifi and PubSubClient libraries allow connection to the Wifi Network and the MQTT broker. The `pb_common.h`, `pb.h` and `pb_encode.h` libraries are used for encoding the messages sent from the Anchor up to the MQTT broker in a simplified format for easy packing and unpacking. These message structures are found in `message.pb.h` and `time.h` respectively. The DW1000Ng libraries are used exclusively for UWB communication from Anchor to Tag.

### Modifiable Variables

Next we initialise the variables that are to be input. ***Refer to Note above***.

```C
// Wifi Connection Credentials
#define WIFI_SSID "WIFI NAME"
#define WIFI_PASSWORD "WIFI PASSWORD"

//MQTT Broker credentials
#define MQTT_BROKER "BROKER IP ADDRESS"
#define MQTT_PORT 1883
#define MQTT_USERNAME ""
#define MQTT_PASSWORD ""

//Unique Anchor ID with 16 characters from 0-9, A-F
#define ANCHOR_ID "UNIQUE ANCHOR ID"

//To check if vals are true or false
char detach[3] = "01" ;

//MQTT BROKER TOPICS
const char *topicAlive = "ALIVE TOPIC";
const char *topicData = "ANCHOR DATA TOPIC";

//Global Clock 
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 28800; // 8 hrs in seconds, since SG is UTC +8:00
const int   daylightOffset_sec = 0; // SG has no daylight savings time

//Initialisation of Wifi Client
WiFiClient espClient;
PubSubClient client(espClient);
```

There are multiple variables to be set including the ones mentioned above as well as the Topics that the ESP32 publishes to which includes both the Alive and Anchor Data topics. Ensure that they match those of the MQTT broker to prevent any errors.

### Structures, Configurations and Declarations

```C
//Tag Data Structure
typedef struct {
    char tagId[32]; // Assuming a max tag ID length of 31 chars + null terminator
    message_TagData tagData; // The protobuf structure for tag data
} TagEntry;

//SPI clock setup
#if defined(ESP8266)
const uint8_t PIN_SS = 5;
#else
const uint8_t PIN_SS = 5; // spi select pin
const uint8_t PIN_RST = 35;
#endif

// Position data struct
typedef struct Position {
    double x;
    double y;
} Position;

//Preliminary position For muti anchor situations
Position position_self = {0,0};
Position position_B = {3,0};
Position position_C = {3,2.5};

double range_self;
double range_B;
double range_C;
int i;

boolean received_B = false;

//Recieved Tag data variable initialisation
byte target_eui[8];
byte tag_shortAddress[] = {0x04, 0x00};

byte anchor_b[] = {0x02, 0x00};
uint16_t next_anchor = 2;
byte anchor_c[] = {0x03, 0x00};

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

frame_filtering_configuration_t ANCHOR_FRAME_FILTER_CONFIG = {
    false,
    false,
    true,
    false,
    false,
    false,
    false,
    true /* This allows blink frames */
};
```

These following lines of code initialise different structures and configurations that will be used in the later parts of the code especially during the ranging process.

We next initialise all the functions that we will be using during the Anchor's function which will be defined below.

```C
// DECLARATIONS
//-----------------------------------
void setupWiFi();
void checkWiFiConnection();
void setupMQTT();
void checkMQTTConnection();
void mqttCallback(char* topic, byte* message, unsigned int length);
bool reconnectMQTT();

//--DW1000RangingFunctions--
void RangingFunction();
void updateTagEntry(const char* tagId, bool exists, bool batteryLow, float range);
void calculatePosition(double &x, double &y);
char* bytesToHexString(const uint8_t data[], uint16_t data_size);

void sendAnchorAlive();
bool encode_anchor_id(pb_ostream_t *stream, const pb_field_t *field, void *const *arg);

void sendAnchorData();
bool encode_tags(pb_ostream_t *stream, const pb_field_t *field, void *const *arg);
```

### Setup and Loop

Then we initialise the setup loop which runs once before the loop begins executing. Most of what occurs in the setup loop is the initialisation of configurations both in the Wifi and UWB modules as well as establishing serial communication on the PC for testing.

```C
void setup() {
  // Set software serial baud to 115200;
  Serial.begin(115200);
  delay(1000);

  Serial.println(F("### DW1000Ng-arduino-ranging-anchorMain ###"));

    // initialize the driver
    #if defined(ESP8266)
    DW1000Ng::initializeNoInterrupt(PIN_SS);
    #else
    DW1000Ng::initializeNoInterrupt(PIN_SS, PIN_RST);
    #endif
    Serial.println(F("DW1000Ng initialized ..."));


    // general configuration
    DW1000Ng::applyConfiguration(DEFAULT_CONFIG);
    DW1000Ng::enableFrameFiltering(ANCHOR_FRAME_FILTER_CONFIG);
    DW1000Ng::setEUI(ANCHOR_ID);
    DW1000Ng::setPreambleDetectionTimeout(64);
    DW1000Ng::setSfdDetectionTimeout(273);
    DW1000Ng::setReceiveFrameWaitTimeoutPeriod(5000);
    DW1000Ng::setNetworkId(RTLS_APP_ID);
    DW1000Ng::setDeviceAddress(1);
    DW1000Ng::setAntennaDelay(16436);
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

  //Setup and connect to Wifi and broker
  setupWiFi();
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  setupMQTT();
}
```

Once the setup is run then the Loop runs constantly running the few functions we have setup for it.

```C
void loop() {
  checkWiFiConnection();
  checkMQTTConnection();
  sendAnchorAlive();
  RangingFunction();
}
```

Now we will define all the functions used in both setup and the loop.

### Helper functions

#### Setup Wifi

The setup wifi function connects the esp32 to the Wifi Network with the credentials that you fill in above. It will attempt to connect the Wifi until the connection is established.

```C
void setupWiFi() {
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  while(WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.println("Setup: Connecting to WiFi...");
  }
  Serial.println("Setup: Connected to the WiFi network");
}
```

#### Check Wifi Connection

This function recurs every 10 seconds to ensure that the esp32 is still connected to the Wifi Network

``` C
void checkWiFiConnection() {
  static unsigned long lastCheckTime = 0;
  unsigned long currentTime = millis();
  const unsigned long retryInterval = 10000; // retry every 10s

  // Only attempt to check every 10s
  if (currentTime - lastCheckTime >= retryInterval) {
    lastCheckTime = currentTime;
    if (WiFi.status() != WL_CONNECTED) {
      Serial.println("CheckWiFi: WiFi disconnected. Reconnecting...");
      WiFi.disconnect();
      WiFi.reconnect();
    } else {
      Serial.println("CheckWiFi: WiFi still connected.");
    }
  }
}
```

#### SetupMQTT and MQTT Callback

These establish connection the MQTT broker and ensure that messages can be sent to the specified topics.

```C
void setupMQTT() {
  client.setServer(MQTT_BROKER, MQTT_PORT);
  client.setCallback(mqttCallback);
  Serial.println("Setup: preparing MQTT client...");
}

void mqttCallback(char* topic, byte* message, unsigned int length) {
  Serial.print("Message arrived: [");
  Serial.print(topic);
  Serial.print("]: ");
  for (unsigned int i = 0; i < length; i++) {
    Serial.print((char) message[i]);
  }
  Serial.println();
  Serial.println("-----------------------");
}
```

#### CheckMQTTConnection

This function also recurs every 10 seconds to ensure MQTT conncetion is maintained

``` C
void checkMQTTConnection() {
  // If connected, handle incoming messages with client.loop()
  if (client.connected()) {
    client.loop();
  } else {
    // If not connected, reconnect
    static unsigned long lastReconnectAttempt = 0;
    unsigned long currentTime = millis();
    if (currentTime - lastReconnectAttempt > 5000) {
      lastReconnectAttempt = currentTime;
      if (reconnectMQTT()) {
        lastReconnectAttempt = 0;
      }
    }
  }
}
```

#### ReconnectMQTT

This function ensures the reconnection to the MQTT broker in case of disconnection. In case of failure it returns the failure state for easier troubleshooting.

```C
bool reconnectMQTT() {
  // Generate a client ID
  String clientId = "esp32-client-";
  clientId += String(WiFi.macAddress());


  // Attempt to connect
  Serial.printf("The client %s tries to connect to MQTT broker\n", clientId.c_str());
  if (client.connect(clientId.c_str(), MQTT_USERNAME, MQTT_PASSWORD)) {
    Serial.println("Connected to MQTT Broker!");
    // Subscribe to topics here, if needed
    // client.subscribe("some/topic"); // no need as we are not getting anything
    return true;
  } else {
    Serial.print("Reconnect failed, state: ");
    Serial.print(client.state());
    return false;
  }
}
```

### SendAnchorAlive

This function takes the anchor credentials - The ID, the timestamp (current time) and a boolean value for whether the timestamp exists. It then packages it using the message format for AnchorAlive. This is then published to the AnchorAlive topic every 10s.

```C
void sendAnchorAlive() {
  static unsigned long lastCheckTime1 = 0;
  unsigned long currentTime1 = millis();
  const unsigned long retryInterval1 = 2000; // retry every 10s

  // Only attempt to check every 10s
  if (currentTime1 - lastCheckTime1 >= retryInterval1) {
    lastCheckTime1 = currentTime1;
  // Initialise AnchorAlive message
  message_AnchorAlive aliveMessage = message_AnchorAlive_init_zero;

  // Populate the message
  aliveMessage.anchor_id.funcs.encode = encode_anchor_id;
  aliveMessage.has_timestamp = true; // Indicate that the timestamp field will be used
  
  time_t now;
  time(&now);
  aliveMessage.timestamp.seconds = now;
  aliveMessage.timestamp.nanos = 0; // Nanoseconds part is not needed for this use case
  Serial.print("time: ");
  Serial.print(now);
  Serial.println();


  // Encode the message
  uint8_t buffer[128]; // Declare buffer
  pb_ostream_t stream = pb_ostream_from_buffer(buffer, sizeof(buffer)); // Initialise stream with this buffer
  aliveMessage.anchor_id.funcs.encode = encode_anchor_id; // Set callback for anchor id

  bool status = pb_encode(&stream, message_AnchorAlive_fields, &aliveMessage);
  if (!status) {
    Serial.print("Encoding AnchorAlive failed: ");
    Serial.println(PB_GET_ERROR(&stream));
    return;
  }

  // Print raw bytes in buffer before publishing
  Serial.println("Raw alive: ");
  for (int i = 0; i < stream.bytes_written; i++) {
      Serial.printf("%02X ", buffer[i]);
  }
  Serial.println();

  //Publish the message
  if (!client.publish(topicAlive, buffer, stream.bytes_written)) {
    Serial.println("Failed to publish message");
  } else {
      Serial.println("AnchorAlive message published successfully");
  }
}}
```

In order to encode the anchor ID, which is input as a string we utilise yet another function.

```C
bool encode_anchor_id(pb_ostream_t *stream, const pb_field_t *field, void *const *arg) {
    const char *str = ANCHOR_ID;
    return pb_encode_tag_for_field(stream, field) &&
           pb_encode_string(stream, (uint8_t*)str, strlen(str));
}
```

### SendAnchorData

This function compiles the Anchor credentials similar to send AnchorAlive - including the timestamp and packs it along with the data recieved from the Tag. It then publishes to the AnchorData topic on the MQTT broker.

```C
void sendAnchorData() {
  // Initialise AnchorData message
  message_AnchorData dataMessage = message_AnchorData_init_zero;

  // Populate the message
  dataMessage.anchor_id.funcs.encode = encode_anchor_id; // Assuming encode_anchor_id is your callback function for encoding the anchor ID
  dataMessage.has_timestamp = true;
  time_t now;
  time(&now);
  dataMessage.timestamp.seconds = now; // Current time in seconds
  // struct timespec ts;
  // clock_gettime(CLOCK_REALTIME, &ts);
  // dataMessage.timestamp.nanos = ts.tv_nsec;
  dataMessage.timestamp.nanos = 0; // Nanoseconds part, typically not needed
  Serial.print("time: ");
  Serial.print(now);
  Serial.println();

  // Encode the message
  uint8_t buffer[1024]; // Declare buffer
  pb_ostream_t stream = pb_ostream_from_buffer(buffer, sizeof(buffer)); // Initialise stream with this buffer
  dataMessage.anchor_id.funcs.encode = encode_anchor_id; // Set callback for anchor id
  dataMessage.tags.funcs.encode = encode_tags; // Set callback for tags
  bool status = pb_encode(&stream, message_AnchorData_fields, &dataMessage);
  if (!status) {
    Serial.print("Encoding AnchorData failed: ");
    Serial.println(PB_GET_ERROR(&stream));
    return;
  }

  // Print raw bytes in buffer before publishing
  Serial.println("Raw data: ");
  for (int i = 0; i < stream.bytes_written; i++) {
      Serial.printf("%02X ", buffer[i]);
  }
  Serial.println();

  // Publish the message
  if(!client.publish(topicData, buffer, stream.bytes_written)) {
    Serial.println("Failed to publish message");
  } else {
    Serial.println("AnchorData message published successfully");
  }}
  ```

  This function uses two helper functions, one to encode strings - to help encode anchor ID and tag ID.

  ```C

  bool encode_string(pb_ostream_t *stream, const pb_field_t *field, void *const *arg) {
    const char *str = (const char*)*arg;
    if (!pb_encode_tag_for_field(stream, field)) return false;
    return pb_encode_string(stream, (const uint8_t*)str, strlen(str));
}
```

The tag encoder takes the tag data that is recieved by the anchor and converts it into a hashmap format required for packaging using the format and sending it to the AnchorData topic. This allows for the Tag ID as well as the distance calculated by the Two Way Ranging function to be packaged into the right format necessary for packaging and transmission.

```C
bool encode_tags(pb_ostream_t *stream, const pb_field_t *field, void *const *arg) {
    for (size_t i = 0; i < numTags; i++) {
        TagEntry* entry = &tagsArray[i];


        // Create a temporary message_AnchorData_TagsEntry to hold our data
        message_AnchorData_TagsEntry tagEntry = message_AnchorData_TagsEntry_init_zero;

        // Prepare the key (tagId) and value (tagData)
        tagEntry.key.funcs.encode = &encode_string;
        tagEntry.key.arg = entry->tagId;
        tagEntry.value = entry->tagData;
        tagEntry.has_value = true; // Indicate that value field is populated

        // Encode the tag entry as a submessage
        if (!pb_encode_tag_for_field(stream, field)) return false;
        if (!pb_encode_submessage(stream, message_AnchorData_TagsEntry_fields, &tagEntry)) return false;
    }
    return true;
}
```

The above two functions ensure that the data is packaged and sent in the right format to the MQTT broker.

### Ranging Function

This function implents Two Way Ranging to calculate the distance between the Tag and Anchor. Once the Tag is recognised and the distance is calculated this data is packaged using the encode tag functions and sent to the MQTT broker using the sendAnchorData function.

Two Way Ranging can be explained as follows:

![TWR_scheme.png](/img/user/Attachments/hardware/TWR_scheme.png)

*Fig 2: TWR Diagram (Sewio, 2024)*

The Tag sends a ranging poll at the start, the Anchor on recieving the tag notes the timestamp and sends a response to the Tag. The Tag once it recieves the anchor's message notes the timestamp and relays it back to the anchor. Using the formula described in the picture above and subtracting the time required for the computation, the time of travel is calculated. This is then multiplied with the speed of light to get the distance between the Tag and Anchor.

This data, along with tag information as well the anchor information is packed and sent to the broker.

```C
void RangingFunction(){
  if(DW1000NgRTLS::receiveFrame()){
        size_t recv_len = DW1000Ng::getReceivedDataLength();
        byte recv_data[recv_len];
        DW1000Ng::getReceivedData(recv_data, recv_len);
        
        if(recv_data[0] == BLINK) {
            byte EUI_STRING[8];
            for (int i = 0; i < 8; i++){
              EUI_STRING[i]= recv_data[9 - i];
            }
            char* hexString = bytesToHexString(EUI_STRING, 8);
            Serial.println("Hexadecimal String: " );
            Serial.print( hexString );

            DW1000NgRTLS::transmitRangingInitiation(&recv_data[2], tag_shortAddress);
            //Serial.println(tag_shortAddress[0], HEX);
            //Serial.println(tag_shortAddress[1], HEX);
            DW1000NgRTLS::waitForTransmission();

            RangeAcceptResult result = DW1000NgRTLS::anchorRangeAccept(NextActivity::RANGING_CONFIRM, next_anchor);
            if(!result.success) {
              free(hexString);
              return;
            }
            range_self = result.range;

            String rangeString = "Range: "; 
            rangeString += range_self; 
            rangeString += " m \t RX power: ";
            rangeString += DW1000Ng::getReceivePower(); 
            rangeString += " dBm";
            delay(1000);
            //Serial.println(rangeString);
            //Setting up to send to middleware, TAGID to char
            // Identifying states
            char state[2];
            Serial.print("detach:");
            Serial.println(state);
            sprintf(state , "%02X", EUI_STRING[7]);
            Serial.print("detach:");
            Serial.println(state);
            bool battery = false;
            bool t_exists = false;
            if (state[1] == detach[1]){
              battery = false;
              t_exists = true;
            }
            Serial.print("detach:");
            Serial.println(t_exists);
            if ( range_self> 0.00001 && range_self <= 100){
            //updateTagEntry(hexString, t_exists, battery, range_self);
            strncpy(tagsArray[0].tagId, hexString, sizeof(tagsArray[0].tagId) - 1);
            tagsArray[0].tagId[sizeof(tagsArray[0].tagId) - 1] = '\0'; // Ensure null termination
            tagsArray[0].tagData.detached = t_exists; // Assuming 'detached' is the opposite of 'exists'
            tagsArray[0].tagData.battery_low = battery; 
            tagsArray[0].tagData.distance = range_self;
            sendAnchorData();}
            // Don't forget to free the allocated memory
            free(hexString);

        } else if(recv_data[9] == 0x60) {
            double range = static_cast<double>(DW1000NgUtils::bytesAsValue(&recv_data[10],2) / 1000.0);
            String rangeReportString = "Range from: "; rangeReportString += recv_data[7];
            rangeReportString += " = "; rangeReportString += range;
            delay(500);
            Serial.println(rangeReportString);
            if(received_B == false && recv_data[7] == anchor_b[0] && recv_data[8] == anchor_b[1]) {
                range_B = range;
                received_B = true;
            } else if(received_B == true && recv_data[7] == anchor_c[0] && recv_data[8] == anchor_c[1]){
                range_C = range;
                double x,y;
                calculatePosition(x,y);
                String positioning = "Found position - x: ";
                positioning += x; positioning +=" y: ";
                positioning += y;
                delay(500);
                Serial.println(positioning);
                received_B = false;
            } else {
                received_B = false;
            }
        }
    }
}
```

This function acts as the main function that runs when the tag is in motion and provides real time updates to the broker which sends it upward along the stream. This function uses two helper functions calculatePosition  and BytestoHexString.

```C
void calculatePosition(double &x, double &y) {


    /* This gives for granted that the z plane is the same for anchor and tags */
    double A = ( (-2*position_self.x) + (2*position_B.x) );
    double B = ( (-2*position_self.y) + (2*position_B.y) );
    double C = (range_self*range_self) - (range_B*range_B) - (position_self.x*position_self.x) + (position_B.x*position_B.x) - (position_self.y*position_self.y) + (position_B.y*position_B.y);
    double D = ( (-2*position_B.x) + (2*position_C.x) );
    double E = ( (-2*position_B.y) + (2*position_C.y) );
    double F = (range_B*range_B) - (range_C*range_C) - (position_B.x*position_B.x) + (position_C.x*position_C.x) - (position_B.y*position_B.y) + (position_C.y*position_C.y);

    x = (C*E-F*B) / (E*A-B*D);
    y = (C*D-A*F) / (B*D-A*E);
}
```

Bytes to hexstring extracts the Tag ID from transmitted data and converts it to a string.

``` C

char* bytesToHexString(const uint8_t data[], uint16_t data_size) {
    // Calculate the length of the resulting hex string, each byte contributes 2 characters
    size_t resultLength = data_size * 2;

    // Allocate memory for the character array (don't forget space for the null terminator)
    char* resultCharArray = (char*)malloc((resultLength + 1) * sizeof(char));

    // Check if memory allocation was successful
    if (resultCharArray == NULL) {
        fprintf(stderr, "Memory allocation failed.\n");
        exit(EXIT_FAILURE);
    }

    // Loop through the data and convert each byte to a two-character hexadecimal representation
    for (int i = 0; i < data_size; i++) {
        // Use snprintf to safely format the hex string into the resultCharArray
        snprintf(resultCharArray + (i * 2), 3, "%02X", data[i]);
    }

    // Add the null terminator at the end
    resultCharArray[resultLength] = '\0';

    return resultCharArray;
}

```

### Uploading the code

Compile all the code described above. The arduino `.ino` file can be found in the following [link](https://github.com/S32-PAMS/PAMS-Hardware/tree/main/anchor_code). Plug the ESP32 to the computer and open the Arduino Code Editor. Select the board that you are using as well as the port, for this project we used the ESP32 Devkit V1 model. Compile and upload the code to the board. While the ESP32 is connected to the device you are using, you can open the Serial Monitor on the Arduino IDE for troubleshooting.

## Security Additions

While the above code allows for basic anchor functionality, the anchor needs to be equipped with greater security features. Some of these features have already been implemented into the code, mainly to enable TLS communication between the Anchor and MQTT broker. The TWR communication is already more or less protected from relay and replay attacks due to most of the calculations being conducted using timestamps.

### TLS

For enabling TLS communication on the anchor, first certificates have to be generated for the anchor and compiled as `.c` files and all of these must be accessible from a `.h` header file. For the purposes of our testing our generated certificates `anchor_1_key.c`, `anchor_1_cert.c` and `rootCA.c` with their corresponding header files. We include the following header files into the code.

``` C
#include "anchor_1_key.h"
#include "anchor_1_cert.h"
#include "rootCA_cert.h"
```

This will load the certs into the code for use. Certain parameters will have be changed to enable TLS.

>**Note : MQTT_PORT needs to be changed to 8883**

```C
#define MQTT_PORT 8883
```

In addition to this we modify the setupMQTT function to provide the capability to connect to the MQTT broker with TLS security enabled.

```C
// -- MQTT --
void setupMQTT() {
  client.setServer(MQTT_BROKER, MQTT_PORT);
  
  Serial.println("Setup: preparing MQTT client...");
  
  // Load certificates into WiFiClientSecure object
  espClient.setCACert(_opt_rootca_local_rootCA_cert_der, _opt_rootca_local_rootCA_cert_der_len); // Add your CA certificate
  espClient.setCertificate(anchor_1_cert_der , anchor_1_cert_der_len); // Add your Client certificate
  espClient.setPrivateKey(anchor_1_key_der,anchor_1_key_der_len ); // Add your Private Key
    
  Serial.println("CA cert identified");
  client.setCallback(mqttCallback);
  
}
```

## Conclusion

Once the anchors are completely set up and configured to the system network and the [[Server/MQTT Broker\|MQTT Broker]], the anchor code can be uploaded and tested for working. In order to see if the code is working the ESP32 of the anchor, you can view the Serial Monitor to see the outputs of the anchor.

## References

Sewio. (2024, January 18). Analyzing decawave two way ranging (TWR). Sewio RTLS. https://www.sewio.net/uwb-sniffer/analyzing-decawave-two-way-ranging-twr/

F-Army. (n.d.). F-Army/Arduino-DW1000-NG: Arduino driver and library to use decawave’s DW1000 IC and relative modules. GitHub. https://github.com/F-Army/arduino-dw1000-ng

Tao, D. (2023, June 10). MQTT on esp32: A beginner’s guide. www.emqx.com. https://www.emqx.com/en/blog/esp32-connects-to-the-free-public-mqtt-broker

DFRobot. (2019, January 25). Esp32 / esp8266 Arduino Tutorial:5. protocol buffers: Messages with strings. https://www.dfrobot.com/blog-1162.html
