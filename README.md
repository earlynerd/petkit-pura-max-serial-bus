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

sample packet
`0x5AA5130218FF80000000000002006400033EA0`
`0x5AA5`: packet header. packets always begin with this header
`0x13`: packet total length in bytes including the header
`0x02`: packet type. `02` is the command that controls motors
`0x18`: sequence counter
`0xFF`: always `FF`, no function besides marking the beginning of payload
`0x80`: the most significant bit in this byte is definitely some sort of flag, and the lower 7 bits may also be flags. TBD.
`0x00000000000200640003`: payload data. exact meaning TBD. for this motor command, it should indicate which motor, how fast, how much current, etc.
`0x3EA0`: CRC-16 CCITT with `0xFFFF` seed. sent Least significant byte first (the feeder sent this in the reverse order). 