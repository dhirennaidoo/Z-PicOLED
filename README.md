# Z-PicOLED
Pi Pico, Waveshare OLED2.23, Pico-CANB

Parts List:
https://www.waveshare.com/wiki/Pico-OLED-2.23

https://www.waveshare.com/wiki/Pico-CAN-B

https://www.raspberrypi.com/products/raspberry-pi-pico/

Currently, filtering is disabled (CANB board will evaluate all CANbus identifiers) but it is, of course, capable of filtering if the correct lines are changed.
The Init function can be supplied with an identifier which will, in turn, set up the CANB board with the appropriate mask.
