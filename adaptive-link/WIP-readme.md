## RADIO - oipcv2.lua
Put this script on your EdgeTX radio /SCRIPTS/FUNCTIONS/ folder.

Setup lua script support on either UBS/VCP or aux1. I use a BLE chip connected to aux1. You may have to adapt the baudrate in the init() function. Default 115200.

Setup the function script to run. It will check serial port for incoming data every 200ms. If data is sent between two slashes in range -1024 to 1024, it will update GVar1 in flightmode0 with this number.
For radxa, the USB console need to be set in correct mode: stty -F /dev/ttyACM0 115200 raw -echo -echoe -echok -echoctl -echoke

In our limited testing, it seems the serial port randomly shuts down after a while on radxa. dont know why (Source: greg)

"echo /-1000/ > /dev/tty/ACM0" will set Gvar1 to value -1000 and output NOK+-1000 to the serial interface.
"echo /hello/ > /dev/tty/ACM0" will do nothing and return "NAN+hello"
The code is a bit dirty, have fun :)

## GROUND STATION - openipc_controlv2.py
Run using "./openipc_control2.py -i wfb_data -o /dev/tty/ACM0"
We have hijacked wfb-cli to write out stats to a file (wfb_stats), which  our control-link.py python script reads back, parse and send over serial link. Dirty code here as well :-)

In order for the feedback to work, it means "wfb-cli gs" must be running in the background. See cli.py line 189-194 for the hijack.

Data is output to wfb_data, session_data, packets_data. Feel free to change and adjust according to your needs.
If someone feels to help, i think it would be great to have a msgpack parser reading directly from default port 8002 instead of writing/reading to file.


## EXPERIMENTAL WFB_NG Link
put socat in /usr/bin (ssc30kq / ssc338q)
open two shells on your drone
Run in first shell:
```
wfb_rx -c 127.0.0.1 -u 5000 -K /etc/drone.key -p 1 -i 7669207 wlan0
```
Run in second shell
```
socat UDP-RECV:5000 STDOUT | /bin/sh
```

Now go to your groundstation, open two shells:
In first shell run:
```
sudo wfb_tx -p 1 -u 5000 -K /etc/gs.key -R 456000 -B20 -M 2 -S 1 -L 1 -G short -k 3 -n 4 -i 7669207 -f data wlan1
```
In second shell
```
socat - UDP-DATAGRAM:127.0.0.1:5000,sp=5000
```
you can do "echo Hello" just to see some output confirmed (Hello).
or try out 
```
/etc/init.d/S95majestic restart
```
This should restart majestic.
