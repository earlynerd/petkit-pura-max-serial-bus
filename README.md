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

# Packet structure

sample packet<br/>
`0x5AA5130218FF80000000000002006400033EA0`<br/>
`0x5AA5`: packet header. packets always begin with this header<br/>
`0x13`: packet total length in bytes including the header<br/>
`0x02`: packet type. `02` is the command that controls motors<br/>
`0x18`: sequence counter<br/>
`0xFF`: always `FF`, no function besides marking the beginning of payload<br/>
`0x80`: the most significant bit in this byte is definitely some sort of flag, and the lower 7 bits may also be flags. TBD.<br/>
`0x00000000000200640003`: payload data. exact meaning TBD. for this motor command, it should indicate which motor, how fast, how much current, etc.<br/>
`0x3EA0`: CRC-16 CCITT with `0xFFFF` seed. sent Least significant byte first (the feeder sent this in the reverse order). <br/>

commands of the same type can be different langths, but commands of matching type and matching "unknown" field are always the same length. it must be a node or address or subcommand type field. 

# Bus Dynamics

Every ESP32 command must be followed up with an acknowledgement from the ISD. But the ISD also initiates updates to the ESP32 without being queried. When a sensor changes there is a corresponding update.  <br/>
I thin k the regular updates are just a list of the status of all the sensors. 

# ESP32 init/boot commands

always starts with command type 9. Then, then the ISD responds, follows up with this burst of subsequent initialization commands. <br/>
```
ESP32, n:32,time:111730, packet type:9, length:9, seq:4, unkbit:1, unk:0, crc:5F82, calccrc:5F82, valid:1, Raw:0x5AA5090904FF80825F
"5AA50C0902FF80000B74B328" <- Expected ISD response
ESP32, n:33,time:116693, packet type:2, length:19, seq:3, unkbit:0, unk:0, crc:1536, calccrc:1536, valid:1, Raw:0x5AA5130203FF00020000000002006400003615
ESP32, n:34,time:116733, packet type:2, length:19, seq:4, unkbit:0, unk:1, crc:0777, calccrc:0777, valid:1, Raw:0x5AA5130204FF01020000000002006400007707
ESP32, n:35,time:116734, packet type:12, length:17, seq:4, unkbit:0, unk:0, crc:D2B7, calccrc:D2B7, valid:1, Raw:0x5AA5110C04FF000AC80F0A0A0A0500B7D2
ESP32, n:36,time:116734, packet type:12, length:17, seq:5, unkbit:0, unk:1, crc:4948, calccrc:4948, valid:1, Raw:0x5AA5110C05FF01C800C800C800C8004849
ESP32, n:37,time:116775, packet type:12, length:16, seq:6, unkbit:0, unk:2, crc:1C74, calccrc:1C74, valid:1, Raw:0x5AA5100C06FF02FC011F004C7001741C
ESP32, n:38,time:116775, packet type:13, length:16, seq:3, unkbit:0, unk:0, crc:EAF9, calccrc:EAF9, valid:1, Raw:0x5AA5100D03FF0002FF0A001E0006F9EA
ESP32, n:39,time:116816, packet type:13, length:16, seq:4, unkbit:0, unk:1, crc:B1D0, calccrc:B1D0, valid:1, Raw:0x5AA5100D04FF0102FF0A001E0006D0B1
ESP32, n:40,time:116816, packet type:4, length:20, seq:3, unkbit:0, unk:0, crc:E97D, calccrc:E97D, valid:1, Raw:0x5AA5140403FF00000000DC05C8001E0014007DE9
ESP32, n:41,time:116857, packet type:4, length:20, seq:4, unkbit:0, unk:1, crc:8F85, calccrc:8F85, valid:1, Raw:0x5AA5140404FF010000001405C8001E001400858F
ESP32, n:42,time:116857, packet type:0, length:10, seq:2, unkbit:0, unk:0, crc:7387, calccrc:7387, valid:1, Raw:0x5AA50A0002FF00008773
ESP32, n:43,time:116898, packet type:1, length:12, seq:3, unkbit:0, unk:0, crc:482F, calccrc:482F, valid:1, Raw:0x5AA50C0103FF000A0A0A2F48
ESP32, n:44,time:116898, packet type:7, length:14, seq:2, unkbit:0, unk:0, crc:DFCA, calccrc:DFCA, valid:1, Raw:0x5AA50E0702FF00050A000A00CADF
```
