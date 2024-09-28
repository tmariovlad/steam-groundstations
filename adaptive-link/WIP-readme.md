# New Stuff!
## Groundstation: /etc/wifibroadcast.cfg
```
[common]
log_interval = 300 # To increase the updates from wfb_ng where we calculate the data sent to the drone.


## Replace the tunnel (which you probably do not use??) with adaptive_link stats. It will create a listening port at 9999 which our python script connects to and send data to the drone.
[gs]
streams = [{'name': 'video',   'stream_rx': 0x00, 'stream_tx': None, 'service_type': 'udp_direct_rx',  'profiles': ['base', 'gs_base', 'video', 'gs_video']},
           {'name': 'mavlink', 'stream_rx': 0x10, 'stream_tx': 0x90, 'service_type': 'mavlink', 'profiles': ['base', 'gs_base', 'mavlink', 'gs_mavlink']},
           {'name': 'adaptive_link', 'stream_rx': None, 'stream_tx': 0x30, 'service_type': 'udp_proxy', 'profiles': ['base', 'radio_base'], 'peer': 'listen://127.0.0.1:9999', 'keypair': '
           ]
```
## Groundstation: Install python script, generate a config file and change it
install adaptive_link.py in /usr/bin
for now, run it manually with "adaptive-link.py --verbose" once to generate config.ini. Ctrl+C to exit. The config will be placed in the working directory of adaptive_link.py (/usr/bin in this case)
Change the udp port to match the port setup in wfb_ng, 9999 in this case.
Run adaptive_link.py --verbose too see output, it should look like this:
![image](https://github.com/user-attachments/assets/d92d295b-473b-4a3e-9fc0-f68c30cf3521)
The warning is shown because there is no data in the buffer, the python scirpt is checking for new data slightly faster than it is provided by wfb_ng.

If everything is working, open a separate window and run wfb-cli gs, it should now be displaying data on the adaptive_link interface:
![image](https://github.com/user-attachments/assets/a767c5de-7245-4b70-9b63-9e2c96967f98)

Add adaptive_link.py to a startup script, this depends on your groundstation setup and I will leave it to you to figure out :-) 
If you want to point to a specific config file you can run it with adaptive_link.py --config /path/to/config.ini

## Drone: /etc/rc.init
You need to install the client application to /usr/bin, and run it once to create the config.cfg. Adapt the config as needed.
You also want to add "adaptive_link &" to /etc/init.rc and start a companion wfb_rx to recieve the data:
wfb_rx -c 127.0.0.1 -u 9999 -K /etc/drone.key -p 48 -i 7669206 wlan0 >/dev/null &
adaptive_link_client --config /usr/bin/config.cfg &

To test it on drone side, start wfb_rx manually:
wfb_rx -c 127.0.0.1 -u 9999 -K /etc/drone.key -p 48 -i 7669206 wlan0

It should output similar to:

![image](https://github.com/user-attachments/assets/38a7a1fb-00c7-40d5-bd3e-87fcfc74111b)

If the link is working, now open a separate shell to youre drone and start adaptive link (adaptive_link_client --config /usr/bin/config.cfg):

![image](https://github.com/user-attachments/assets/21dbc25b-e8be-45f8-8a5b-5cfed14d2b7e)
the binary name is different on the picture, will change it later, usage will be as described in text above.

Now you need to setup you channels.sh to act on the 1000-2000 data being sent to it! 
```
#!/bin/sh

	# Decide MCS, guard interval, FEC, bit-rate, gop size and power based on the range [1000,2000] from radio
	# Need > wfb_tx v24 (with -C 8000 added to /usr/bin/wifibroadcast) and wfb_tx_cmd v24
	# Need msposd (tipoman9) to update local OSD

if [ -e /etc/txprofile ]; then
	. /etc/txprofile
fi

oldProfile=$vidRadioProfile
newTimeStamp=$(date +%s)
oldTimeStamp=$prevTimeStamp
secondsSinceLastChange=$((newTimeStamp-oldTimeStamp))
	
	if [ $2 -eq 999 ] ;then
				
		setGI=long
		setMCS=0
		setFecK=12
		setFecN=15
		setBitrate=3332
		setGop=1.0
		wfbPower=54
		ROIqp="0,0,0,0"		
		newProfile=0

	elif [ $2 -eq 1000 ] ;then
				
		setGI=long
		setMCS=0
		setFecK=12
		setFecN=15
		setBitrate=3333
		setGop=1.0
		wfbPower=48
		ROIqp="0,0,0,0"		
		newProfile=1
		
	elif [ $2 -lt 1300 ];then
		
		setGI=long
		setMCS=1
		setFecK=12
		setFecN=15
		setBitrate=6667
		setGop=1.0
		wfbPower=48
		ROIqp="12,6,6,12"
		newProfile=2

	elif [ $2 -lt 1700 ];then

		setGI=long
		setMCS=2
		setFecK=12
		setFecN=15
		setBitrate=7333
		setGop=1.0
		wfbPower=48
		ROIqp="12,6,6,12"
		newProfile=3


	elif [ $2 -lt 2000 ];then
						
		setGI=long
		setMCS=2
		setFecK=12
		setFecN=15
		setBitrate=7333
		setGop=1.0
		wfbPower=48
		ROIqp="12,6,6,12"
		newProfile=4

	elif [ $2 -eq 2000 ];then

		setGI=short
		setMCS=2
		setFecK=12
		setFecN=15
		setBitrate=7666
		setGop=1.0
		wfbPower=48
		ROIqp="12,6,6,12"
		newProfile=5
	fi

#Decide if it is worth changing or not
profileDifference=$((newProfile - oldProfile))

#if in fallback mode, always hold there a couple of seconds
if [ $oldProfile -eq 0 ] && [ $secondsSinceLastChange -le 3 ] ; then
	exit 1
elif [ $profileDifference -eq 1 ] && [ $secondsSinceLastChange -le 2 ] ; then
	exit 1
elif [ $newProfile -eq $oldProfile ] ;then
	exit 1
fi


# Calculate driver power
setPower=$((wfbPower * 50))


#Decide what order to exectute commands
if [ $newProfile -gt $oldProfile ]; then
###########################################################################	
	# Lower power first
	if [ $prevSetPower -ne $setPower ]; then
		iw dev wlan0 set txpower fixed $setPower
		sleep 0.075
	fi
	
	# Set gopSize
	if [[ "$prevGop" != "$setGop" ]]; then
		#curl localhost/api/v1/set?video0.gopSize=$setGop
		sleep 0.075
	fi
	
	# Raise MCS
	if [ $prevMCS -ne $setMCS ]; then
		wfb_tx_cmd 8000 set_radio -B 20 -G $setGI -S 1 -L 1 -M $setMCS
		sleep 0.075
	fi
	
	# Change FEC
	if [ $prevFecK -ne $setFecK ] || [ $prevFecN -ne $setFecN ]; then
		wfb_tx_cmd 8000 set_fec -k $setFecK -n $setFecN
		sleep 0.075
	fi	
	# Increase bit-rate
	if [ $prevBitrate -ne $setBitrate ]; then
		curl -s "http://localhost/api/v1/set?video0.bitrate=$setBitrate"
		#echo IDR 0 | nc localhost 4000 > /dev/null
		sleep 0.1
	fi
	
	# Change ROIqp
	if [[ "$prevROIqp" != "$ROIqp" ]]; then
		
		#curl localhost/api/v1/set?fpv.roiQp=$ROIqp
		sleep 0.075
	fi
	
elif [ $newProfile -lt $oldProfile ]; then
###############################################################################	
	# Decrease bit-rate first
	if [ $prevBitrate -ne $setBitrate ]; then
		curl -s "http://localhost/api/v1/set?video0.bitrate=$setBitrate"
		#echo IDR 0 | nc localhost 4000 > /dev/null
	fi
	
# Set gopSize
	if [[ "$prevGop" != "$setGop" ]]; then
		#curl localhost/api/v1/set?video0.gopSize=$setGop
		sleep 0.075
	fi
	
	# Lower MCS
	if [ $prevMCS -ne $setMCS ]; then
		wfb_tx_cmd 8000 set_radio -B 20 -G $setGI -S 1 -L 1 -M $setMCS
		sleep 0.075
	fi

	#change FEC
	if [ $prevFecK -ne $setFecK ] || [ $prevFecN -ne $setFecN ]; then
		wfb_tx_cmd 8000 set_fec -k $setFecK -n $setFecN
		sleep 0.075
	fi
	
	# Increase power
	if [ $prevSetPower -ne $setPower ]; then
		iw dev wlan0 set txpower fixed $setPower
		sleep 0.075
	fi
	
	# Change ROIqp
	if [[ "$prevROIqp" != "$ROIqp" ]]; then
		#curl localhost/api/v1/set?fpv.roiQp=$ROIqp
		sleep 0.075
	fi
fi
#############################################################################

# Display stats on msposd
echo "$2 link $secondsSinceLastChange s $setBitrate M:$setMCS $setGI F:$setFecK/$setFecN P:$wfbPower G:$setGop&L30&F28 CPU:&C &Tc" >/tmp/MSPOSD.msg

#Update all profile variables in file
echo -e "vidRadioProfile=$newProfile\nprevGI=$setGI\nprevMCS=$setMCS\nprevFecK=$setFecK\nprevFecN=$setFecN\nprevBitrate=$setBitrate\nprevGop=$setGop\nprevSetPower=$setPower\nprevROIqp=$ROIqp\nprevTimeStamp=$newTimeStamp" >/etc/txprofile

exit 1
```

# Old stuff
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
