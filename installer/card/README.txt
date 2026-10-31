GC3 RTSP firmware: installation
===============================

This card turns a GNCC GC3 into a plain RTSP camera without cloud, app or
account. It first saves a backup of the camera's flash on this card. You can do
this on a closed camera, without any tools.

1. Open settings.txt and enter the name and password of your Wi-Fi.

2. Power the camera off. Put this card into the camera FIRST, then power the
   camera on. A card that goes in after power-on is ignored.

3. Wait, and do not unplug the camera. This takes about 5 minutes:

     red flashes briefly every 4 s    backup and building, about 1 minute
     red blinking fast                writing the firmware, about 1.5 minutes
     red steady, white blinking       new firmware, connecting to your Wi-Fi
     red steady, white steady         done, the camera is on your network

     red and white blinking together  stopped: read RESULT.txt on this card
     only white steady from the start the installer did not run, try again

   The red LED of the GC3 stays on once this firmware runs. That is normal.

4. Power the camera off, take out the card and read these on your computer:

     CAMERA-ACCESS-<serial number>.txt   stream addresses and passwords
     camera-ip.txt                       the address of the camera

   Your stream:  rtsp://camera:PASSWORD@ADDRESS:8554/main

5. Copy the folder backup-<serial number> to your computer. It is your way
   back to the stock firmware. It holds private data of your camera, like the old
   Wi-Fi password and the cloud certificates: never share it.

This card now holds your Wi-Fi password and your camera passwords. Keep it safe, or
delete settings.txt and CAMERA-ACCESS-*.txt once you have noted them.

More cameras: use the same card again, one camera after the other.

Going back to the stock firmware, problems, details:
https://github.com/LarzVader/gncc-gc3-rtsp
