# ESPNOW flooding mesh library 2

Forked from https://github.com/leodesigner/espNowFloodingMeshLibrary2,
in turn forked from https://github.com/arttupii/espNowFloodingMeshLibrary

This fork only fixes a few issues I had with leodesigners library, mainly with esp32-C3.

The example files in here seems a bit outdated... I had better success with leodesigners example project: https://github.com/leodesigner/esp_mesh_pir_sensor

----

**I have also created a Home Assistant Add-on: [EspNow2mqtt](https://github.com/314159-r/esp_mesh_mqtt_gateway)** &emsp; <img src="https://github.com/314159-r/esp_mesh_mqtt_gateway/blob/master/images/home-assistant-logo.png"> \
Providing a gateway for EspNow devices to Home Assistant via MQTT. 
It is a from a fork of the python gateway by [leodesigner](https://github.com/leodesigner/esp_mesh_mqtt_gateway).

----

From leodesigners repo...

Features added/bugfixes since version 2:
- a lot of bugfixes and improvements, battle tested with 15+ nodes
- telemetry stats (optional)
- time sync after bootup is now possible from any node, not only master
- new INSTANT_TIME_SYNC_REQ_ANNONCE internal message for node annoncement
- led blinking support on message receive
- mesh begin can now run without WiFi disconnect (speeds up boot time, important for PIR sensor sleeping nodes)
- new function for sending messages with specific message ID (future support for multi mqtt gataway setup)
- message roundtrip reduced as much as possible to get more reponsive real-time system

Features:
- Maximum number of slave nodes: unlimited
- Number of master nodes: 1
- Master node sends time sync message every 10s to all nodes. (this synchronizes the clocks of the nodes)
- a message cache. If a received packet is already found in the cache --> it will not be retransmitted or handled again
- Every message has a time stamp. If the time stamp is too old (or from the future), the message will be rejected.
- All messages are crypted (AES128)
- Flooding mesh support
- TTL support (time to life)
- ESP32, ESP2866, ESP01
- Battery node support (Battery nodes do not relay messages)
- Request&Reply support
- Each Nodes can communicate with each other
- Ping about 40-60ms
- Nearly instant connection after poweron
- Retransmission support
- Request/Reply support
- Send and pray support (Send a message to all nodes without reply/ack)
- Easy to configure (Set only the same bsid, iv and secred key to all nodes)
- Works on esp-now broadcast
- Arduino


## Flooding mesh network
In this network example ttl must be >= 4
```
               SlaveNode
                   |
                   |         Message from master to BatteryNode
                   |   ---------------------------+
                   |                     ttl=4    |
SlaveNode-------MasterNode-------------SlaveNode  |
                   |                     |        |
                   |                     |        |
                   |                     |        |
                   |                     |        |
               SlaveNode                 |        |
                   |                     |        |
                   |                     |        |
                   |                     |        +------------------------------------------------>
                   |                     | ttl=3         ttl=2              ttl=1
SlaveNode-------SlaveNode-------------SlaveNode-------SlaveNode-------------SlaveNode---------BatteryNode
   |               |                     |
   |               |                     |
   |               |                     |
   |               |                     |
   +-----------SlaveNode-----------------+
```  
## Message headers
```
+---------------------------------------------------------------------------------------+
| AES128 Crypted header (Mesh-header part2)                                             |
|   ---------------------------------------------------------------------------------   |
|   |msgId         | length        |  replyId  |   time stamp  |      data          |   |
|   ---------------------------------------------------------------------------------   |
|       1 byte       1 byte           4 byte       8-byte              0~226 bytes      |
+---------------------------------------------------------------------------------------+
                                                        ^
                                                         \
+---------------------------------------------------------\-----------------------------+
| Mesh-header part 1                                       \                            |
|   ---------------------------------------------------------------------------------   |
|   |bsId         | ttl        |  crc    |        AES128 crypted data               |   |
|   ---------------------------------------------------------------------------------   |
|       3 byte       1 byte       2 byte            0~240 bytes (not 244 because AES    |
|                                                          buffer is 16 byte multiple)  |
+---------------------------------------------------------------------------------------+
                                                                    ^
                                                                     \
+-------------------------------------------------------------------- \---------------+
| Espnow-header                                                        \              |
|   -------------------------------------------------------------------------------   |
|   | Element ID | Length | Organization Identifier | Type | Version |    Body    |   |
|   -------------------------------------------------------------------------------   |
|       1 byte     1 byte            3 bytes         1 byte   1 byte   0~250 bytes    |
|                                                                                     |
+-------------------------------------------------------------------------------------+
                                                                    ^                                                                  
                                                                     \
+---------------------------------------------------------------------\--------------------+
|                                                                      \                   |                                             
| ---------------------------------------------------------------------------------------- |
| |MAC Header | Category Code | Organization Identifier | Vendor Specific Content | FCS  | |
| ---------------------------------------------------------------------------------------- |
|                   1 byte              3 bytes                  7~255 bytes               |
+------------------------------------------------------------------------------------------+
```                                                                                        ´

## Create master node:
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len){
  if(len>0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);

  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_setToMasterRole(true,3); //Set ttl to 3. TIME_SYNC message use this ttl
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message[] = "MASTER HELLO MESSAGE";
    espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 3); //set ttl to 3
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```
## Create slave node:
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE,0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len){
  if(len>0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);

  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_secredkey(secredKey);

  //Ask instant sync from master.
  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
  while(espNowFloodingMesh_isSyncedWithMaster()==false);
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message[] = "SLAVE HELLO MESSAGE";
    espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 3); //set ttl to 3
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```

## Create slave node (Battery):
```c++
#include <EspNowFloodingMesh.h>
#include <time.h>
#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55, 0x66, 0x77, 0x88, 0x99, 0xAA, 0xBB, 0xCC, 0xDD, 0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len) {
  if (len > 0) {
    Serial.println((const char*)data);
  }
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_setToBatteryNode();
}

void loop() {
  static unsigned long m = millis();

  //Ask instant sync from master.
  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  while (espNowFloodingMesh_isSyncedWithMaster() == false);
  char message[] = "SLAVE(12) HELLO MESSAGE";
  espNowFloodingMesh_send((uint8_t*)message, sizeof(message), 0); //set ttl to 3
  espNowFloodingMesh_loop();
  ESP.deepSleep(60000, WAKE_RF_DEFAULT); //Wakeup every minute
}
```
## Send message and get reply:
Send "MARCO" to other nodes
```c++

#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len, uint32_t replyPrt){
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);
  espNowFloodingMesh_setToMasterRole(true,3); //Set ttl to 3.
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
}

void loop() {
  static unsigned long m = millis();
  if(m+5000<millis()) {
    char message2[] = "MARCO";
    espNowFloodingMesh_sendAndHandleReply((uint8_t*)message2, sizeof(message2),3,[](const uint8_t *data, int len){
        if(len>0) { //Handle reply from other node
          Serial.print("Reply: "); //Prinst POLO.
          Serial.println((const char*)data);
        }
    });
    m = millis();
  }
  espNowFloodingMesh_loop();
  delay(10);
}
```
Answer to "MARCO" and send "POLO"
```c++
#include <EspNowFloodingMesh.h>

#define ESP_NOW_CHANNEL 1
//AES 128bit
unsigned char secredKey[] = {0x00,0x11,0x22,0x33,0x44,0x55,0x66,0x77,0x88,0x99,0xAA,0xBB,0xCC,0xDD,0xEE, 0xFF};

void espNowFloodingMeshRecv(const uint8_t *data, int len, uint32_t replyPrt){
  if(len>0) {
    if(replyPrt) { //Reply asked. Send reply
        char m[]="POLO";
        Serial.println((char*)data); //Prints MARCO
        espNowFloodingMesh_sendReply((uint8_t*)m, sizeof(m), 0, replyPrt); //Special function for reply messages. Only the sender gets this message.
    } else {
      //No reply asked... All others messages are handled in here.
    }
  }
}

void setup() {
  Serial.begin(115200);
  //Set device in AP mode to begin with
  espNowFloodingMesh_RecvCB(espNowFloodingMeshRecv);
  espNowFloodingMesh_secredkey(secredKey);
  espNowFloodingMesh_begin(ESP_NOW_CHANNEL);

  espNowFloodingMesh_requestInstantTimeSyncFromMaster();
  espNowFloodingMesh_ErrorDebugCB([](int level, const char *str) {
    Serial.println(str);
  });
  while (espNowFloodingMesh_isSyncedWithMaster() == false);
}

void loop() {
  espNowFloodingMesh_loop();
  delay(10);
}
```
