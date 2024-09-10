# petkit-pura-max-serial-bus
UART serial bus protocol used in Pura Max automatic litterboxes

# Hardware
The Pura Max PCBA hosts an ESP32 and an ISD91230 ARM cortex M0. highly reminiscent of the petkit feeder I have recenbtly taken apart.<br/> 
And, in fact it employes a highly similar but not identical sort of UART communications protocol. <br/>
This repo is my efforts to decode the protocol so that an alternate firmware might be developed for the ESP32, without needing to touch the ISD91230.<br/>
Petkit has not locked down either of the microcontrolelrs and so I have been able to dump the flash for both controllers. <br/>
Some major hints to the structure of the packets can be found simply by running the binary images through "strings", including a list of command/packet types.<br/>
I've also had some success with conversion of the bin file back to an ELF file using ESP32Knife, and then decompile in Ghidra using the svdloader plugin.  <br/>


The ISD91230 appears to eb responsible for motor control, beep/audio, reading 4x half-bridge loadcells via an i2c frontend, reading a pair of ambient light and proximity i2c sensors for cat detection, monitoring 6x hall effect sensors. 
The ESP32 handles all high level tasks and seems to interface directly to the RTCC and the OLED display. 
Theres handy labeled testpoints and headers available onboard. SWD for the ISD, and the UART pins for the ESP32. 

# Packet structure

sample packet<br/>
`0x5AA5130218FF80000000000002006400033EA0`<br/>
`0x5AA5`: packet header. packets always begin with this header<br/>
`0x13`: packet total length in bytes including the header<br/>
`0x02`: packet type. `02` is the command that controls motors<br/>
`0x18`: sequence counter<br/>
`0xFF`: always `FF`, no function besides marking the beginning of payload<br/>
`0x80`: the most significant bit in this byte is definitely some sort of flag, and the lower 7 bits seem to indicate packet subtype or address. packets of the same type can be diferent lengths, but packets of matching type and subtype always same length.<br/>
`0x00000000000200640003`: payload data. exact meaning TBD. The payload data is sent LSByte first.<br/>
`0x3EA0`: CRC-16 CCITT with `0xFFFF` seed. sent Least significant byte first (the feeder sent this in the reverse order). <br/>

# Packet types
snips from the strings.txt output related to command/packet types.

```
cmd:%d__,
0x%02x
Heart Beat Cmd!
SENSOR Cmd!
scale Cmd!
ADCmoto Cmd!
Motor Run Config Cmd! wr:%d
MOT_RUNSTA[%d]
MOT_BASE_CFG
MOT_BASE_RUNSTA
error code cmd!
set error code! wait user to develop!
version code cmd!
set version code! wait user to develop!
reset mcu cmd!
>>>>>>>>>>>>> config data cmd! node: %d_%d %d
get cnfig datas! wait user to develop!
LED CFG! node: %d
output cmd node is over led_max! node: %d
get cnfig datas! wait user to develop! %d
```

```
 recv: MCU -> ESP, Heart beat! %d 
 recv: MCU -> ESP, _sensor_node:%d 
 recv: COMM_CMD_VER!
 recv: reset cmd!
 recv: config datas cmd!
 UART recv cmd is over range! cmd: %d
```

## ISD -> ESP async packet types
`type 1 subtype 0`: I think this is just a heartbeat. always same value?<br/>
`type 1 subtype 1`: I think these contain data from the IR reflective proximity adn ambient light sensors used for cat detection. TBD.<br/>
`type 3`: These are updates about motors. Theyre only sent during an active motor movement in progress<br/>
`type 7`: These packets contain scale/loadcell data. each packet contains multiple samples. The first byte indicates how many samples in the packet followed by 24-bit raw adc readings packed into 32 bit int. <br/>

## ESP -> ISD Command/config packet types
`type 0`: TBD<br/>
`type 1`: TBD. probably configures somethign related to prox sensors <br/>
`type 2`: two observed subtypes, probably for two motors. these are definitely motor control. TBD<br/>
`type 4`: two observed subtypes. TBD<br/>
`type 7`: Sent at Boot. This configures the type 7 async packet comms. `0x05 0A00 0A00` 5 samples per packet, 10Hz sample rate, and one more parameter not yet known. <br/>
`type 12`: three observed subtypes. fucntion TBD<br/>
`type 13`: two observed subtypes. TBD<br/>

# Bus Dynamics

Every ESP32 command must be followed up with an acknowledgement from the ISD. But the ISD also initiates updates to the ESP32 without being queried. When a sensor changes there is a corresponding update.  <br/>

# ESP32 init/boot commands

always starts with command type 9. Then, then the ISD responds, follows up with this burst of subsequent initialization commands. <br/>
```
ESP32, n:32,time:111730, packet type:9, length:9.0, seq:4, flag:1, crc:5F82, valid:1
"5AA50C0902FF80000B74B328" <- Expected ISD response
ESP32, n:33,time:116693, packet type:2.0, length:19, seq:3, flag:0, crc:1536, valid:1, Data: 0x0200 0000 0002 0064 0000
ESP32, n:34,time:116733, packet type:2.1, length:19, seq:4, flag:0, crc:0777, valid:1, Data: 0x0200 0000 0002 0064 0000
ESP32, n:35,time:116734, packet type:12.0, length:17, seq:4, flag:0, crc:D2B7, valid:1, Data: 0x0AC8 0F0A 0A0A 0500
ESP32, n:36,time:116734, packet type:12.1, length:17, seq:5, flag:0, crc:4948, valid:1, Data: 0xC800 C800 C800 C800
ESP32, n:37,time:116775, packet type:12.2, length:16, seq:6, flag:0, crc:1C74, valid:1, Data: 0xFC011 F004C 7001
ESP32, n:38,time:116775, packet type:13.0, length:16, seq:3, flag:0, crc:EAF9, valid:1, Data: 0x02 FF 0A00 1E00 06
ESP32, n:39,time:116816, packet type:13.1, length:16, seq:4, flag:0, crc:B1D0, valid:1, Data: 0x02 FF 0A00 1E00 06
ESP32, n:40,time:116816, packet type:4.0, length:20, seq:3, flag:0, crc:E97D, valid:1, Data: 0x00 0000 DC05 C800 1E00 1400
ESP32, n:41,time:116857, packet type:4.1, length:20, seq:4, flag:0, crc:8F85, valid:1, Data: 0x00 0000 1405 C800 1E00 1400
ESP32, n:42,time:116857, packet type:0.0, length:10, seq:2, flag:0, crc:7387, valid:1, Data: 0x00
ESP32, n:43,time:116898, packet type:1.0, length:12, seq:3, flag:0, crc:482F, valid:1, Data: 0x0A 0A 0A
ESP32, n:44,time:116898, packet type:7.0, length:14, seq:2, flag:0, crc:DFCA, valid:1, Data: 0x05 0A00 0A00
```
