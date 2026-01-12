# ESPHome-LVGL-running-on-a-Guition-JC4827W543
### A proof of concept project to control a ESP32-C6 relay module from HA and/or an ESP32 based 4.3in Display Module (January 2026)  

#### Intro  

This project explores the integration of Home Assistant (HA) with a cheap ESP32-C6 relay module and the Guition JC4827W543C 4.3" LCD Display module.  

My aim was to learn how to use ESPHome LVGL to run YAML on the display module to control the relay module both connected to my test Home Assistant
setup on my home network. My grip of ESPHome is not extensive to be fair making me reliant on searching out example code that I can use and
learn from. Furthermore - a quick examination of the ESPHome LVGL documentation https://esphome.io/components/lvgl/ made me realise that learning LVGL from 
scratch was going to be steep learning curve especially as example code seemed to be sparse!  

A quick search revealed that the Gution Display module is supported https://devices.esphome.io/devices/guition-jc4827543c/ and I was able
to copy/paste a short YAML into my ESPHome environment that runs on my development docker computer. This at least worked and formed the base YAML for my proof of concept project.    

The creation of the YAML for the ESP32-C6 Relay module is covered in a separate project  https://github.com/roadsnail/ESP32-C6-Relay  

This project builds upon that allowing me to control it from HA or from the display module


