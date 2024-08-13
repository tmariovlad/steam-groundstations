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
We have hijacked wfb-cli to write out stats to a file (wfb_stats), which  our control-link.py python script reads back, parse and send over serial link.
In order for the feedback to work, it means "wfb-cli gs" must be running in the background.

see cli.py line 189-194 for the hijack.
Data is output to wfb_data, session_data, packets_data. Feel free to change and adjust according to your needs.
If someone feels to help, i think it would be great to have a msgpack parser reading directly from default port 8002 instead of writing/reading to file.
