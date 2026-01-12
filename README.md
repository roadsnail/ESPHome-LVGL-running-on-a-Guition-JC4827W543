# ESPHome-LVGL-running-on-a-Guition-JC4827W543
A proof of concept project to control a ESP32-C6 relay module from HA and/or an ESP32 based 4.3in Display Module

# ESP32-C6-Relay
 This ESP32-C6 Relay Module (ESP32-C6_Relay_X1_V1.1) is available from various sellers on the Internet. It can be powered and programmed via its USB-C socket.  
 
 It can also be powered from a 7-60V DC supply according to the silk screen printing on the bottom of the module (although I haven't tried this yet).  

 I haven't yet been able to find a schematic for the module so I am unsure what isolation exists between the 7-60V DC input and the USB-C socket.  

 I have programmed this using ESPHome. The initial loading of ESPHome (onto the module) was a pain taking several attempts. I'm guessing this may be due to fairly recent support for the C6 version of ESP32 module. 

 
![ESP32-C6-Relay_Module](https://github.com/user-attachments/assets/a2984527-bf0b-4460-a53b-7726257bcbf4)

![ESP32-C6-Relay_Module_1](https://github.com/user-attachments/assets/2bf053be-5092-455f-a437-f65b2f589947)


### GPIO Pin assisgnments
- GPIO2	LED
- GPIO19	Relay
