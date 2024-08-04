# PixMob Reverse Engineering

## Introduction

The reverse engineering in this repository efforts were primarily performed on a PixMob PALM v2.6r1 (c) 20230629. 

## EEPROM
The EEPROM on my PixMob PALM v2.6r1 (c) 20230629 was marked `24C02`, which is a 2Kbit (256 x 8 bits) I2C chip.

Only the first 88 bytes of the EEPROM are used to store information. The remaining 168 bytes are left blank (0xFF) and not touched by the firmware.

The high-level layout of the EEPROM is as follows (each cell in the diagram is 8 bits / 1 byte):
```
             0x00                0x01                0x02                0x03        
     +-------------------+-------------------+-------------------+-------------------+
0x00 |       magic       |     group sel     |       ????        |       ????        |
     +-------------------+-------------------+-------------------+-------------------+
0x04 |     on start      |       ????        |       ????        |       ????        |
     +-------------------+-------------------+-------------------+-------------------+
0x08 |  group sel 0 id   |  group sel 1 id   |  group sel 2 id   |  group sel 3 id   |
     +-------------------+-------------------+-------------------+-------------------+
0x0C |  group sel 4 id   |  group sel 5 id   |  group sel 6 id   |  group sel 7 id   |
     +-------------------+-------------------+-------------------+-------------------+
0x10 |  profle 0 green   |   profile 0 red   |  profile 0 blue   | profile 0 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x14 |  profle 1 green   |   profile 1 red   |  profile 1 blue   | profile 1 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x18 |  profle 2 green   |   profile 2 red   |  profile 2 blue   | profile 2 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x1C |  profle 3 green   |   profile 3 red   |  profile 3 blue   | profile 3 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x20 |  profle 4 green   |   profile 4 red   |  profile 4 blue   | profile 4 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x24 |  profle 5 green   |   profile 5 red   |  profile 5 blue   | profile 5 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x28 |  profle 6 green   |   profile 6 red   |  profile 6 blue   | profile 6 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x2C |  profle 7 green   |   profile 7 red   |  profile 7 blue   | profile 7 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x30 |  profle 8 green   |   profile 8 red   |  profile 8 blue   | profile 8 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x34 |  profle 9 green   |   profile 9 red   |  profile 9 blue   | profile 9 chksum  |
     +-------------------+-------------------+-------------------+-------------------+
0x38 |  profle 10 green  |  profile 10 red   |  profile 10 blue  | profile 10 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x3C |  profle 11 green  |  profile 11 red   |  profile 11 blue  | profile 11 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x40 |  profle 12 green  |  profile 12 red   |  profile 12 blue  | profile 12 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x44 |  profle 13 green  |  profile 13 red   |  profile 13 blue  | profile 13 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x48 |  profle 14 green  |  profile 14 red   |  profile 14 blue  | profile 14 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x4C |  profle 15 green  |  profile 15 red   |  profile 15 blue  | profile 15 chksum |
     +-------------------+-------------------+-------------------+-------------------+
0x50 |   static green    |    static red     |    static blue    |      attack       |
     +-------------------+-------------------+-------------------+-------------------+
0x54 |      sustain      |      release      |   profile range   |       mode        |
     +-------------------+-------------------+-------------------+-------------------+
```

Fields:
* `magic`: A constant  value that  is specific to the firmware running on the PixMob's MCU. An unexpected value here will cause the MCU to set the EEPROM back to factory defaults.
* `group sel`: Indirect [group id](#ir-command-fields-group-id) selection. The lower 3 bits selects one of eight `group sel [0-7] id` fields.
* `group sel [0-7] id`: The lower 5 bits select the [group id](#ir-command-fields-group-id) the PixMob unit is a part of.
* `on start`: When set to 0x11, will enable the "on-start" effect. This effect will cause the PixMob to start displaying colors as soon as it receives power, without needing to receive any IR commands.
* `profile [0-15]`: Defines 16 RGB color profiles that the MCU can access without needing to receive an IR command. The green, red, and blue fields hold the RGB values 0-255. The checksum field is 8 lower bits of the sum of green+red+blue.
* `static green`, `static red`, and `static blue`: Defines a special RGB profile that is used in static mode (see below). There is no checksum for the static RGB profile.
* `attack`, `sustain`, and `release`: Defines the LED fade-in time, hold time, and fade-out time, respectively, in 16-millisecond intervals. Used by the "on-start" effect. For example, a value of 0x1E would be 480ms.
* `profile range`: Defines the lower and upper bounds on the profile id when running in sequential and random mode (see below). The lower 4 bits of this field is the lower bound profile id, while the upper 4 bits of this field is the upper bound profile id.
* `mode`: Mode for the "on-start" effect. If "on-start" effect is not enabled (via the `on start` field), this field is ignored and nothing will happen when the PixMob receives power.
  * Static (0x00): Cycle the same RGB value defined by `static green`, `static red`, and `static blue`. `attack`, `sustain`, and `release` values are honored, but `profile range` is ignored.
  * Sequential (0x02): Cycle the profiles starting at the lower bound profile id defined by `profile range` up to the upper bound profile id defined by `profile range`, then repeat starting back at the lower bound.
  * Random (0x06): Cycle the profiles, each time picking a random profile id between the lower and upper bound profile ids defined by `profile range`.
  * Further use of the remaining possible values are still under investigation.


## MCU RAM

### RGB Config (CFG*)
The 8 bytes at EEPROM address 0x50 to 0x57 collectively form a RGB config struct. Within the MCU register memory (RAM), there exists three instances of the RGB config struct (same format as within the EEPROM table):
* CFG0: The "staging" config. Generally, data from received IR commands are first stored into CFG0.
* CFG1: The "active" config. When the PixMob is ready to display a color, data is generally copied into CFG1 either from CFG0 or CFG2. The PWM cycles are then based off CFG1.
* CFG2: The "storage" config. During initial power-on, data from the EEPROM at 0x50-0x57 are copied into CFG2.


### Global Sustain Time (GST): 
This is a single 8-bit value that is occasionally used to override the sustain timer. Like other timers, each step represents a 16ms time increment.

During initial power-on, the GST is initialized to be 0x1E (480ms). It can then be updated through the [set GST IR command](#ir-command-set-gst).


## IR Commands

### IR Command Header
All PixMob IR commands begin with the same 3-byte header, with the first byte being a magic constant, the second byte being [a computed checksum](#checksum-calculation), and the third byte containing a series of flags that determine the format of the command body.

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x00 |    1    |    0    |    0    |    0    |    0    |    0    |    0    |    0    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x01 |                                   checksum                                    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x02 |    0    |    0    |   ???   |  gsten  |       type        |  /rgb   | onstrt  |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Starting from the command byte located at offset 0x02, only the lower 6 bits of each command byte are used. The upper 2 bits MUST be zero. This is because PixMob uses 6-bit values when transmitting data over DMX for their lighting controllers (likely to reduce the total number of required channels).

Fields:
* `gsten`: At least one of its functions is to override the sustain time from [GST memory](#global-sustain-time-gst). Exact details still under investigation.
* `type`: In conjunction with the total length of the command, determines the format of the command body.
* `/rgb`: Generally set to 1'b0 for commands that result in a color being displayed and set to 1'b1 for other commands. Exact details still under investigation.
* `onstrt`: Set to 1'b1 to enable (or keep enabled) the "on-start" effect. If the "on-start" effect is currently enabled and a command is received with `onstrt=0`, the "on-start" effect will be disabled.


### IR Command: Set Cycle Options
Set color profile cycle options into CFG0 memory. If flags `onstrt=1'b1`, the updated CFG0 memory is also saved to the EEPROM.

Flags: `type=2'b00`, `/rgb=1'b1`, (`onstrt=1'b1` and `gsten=1'b1`) or (`onstrt=1'b0` and `gsten=1'bX`)

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |                    profile range[5:0]                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |           attack            | random  |profile range[7:6] |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |           release           |           sustain           |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Fields:
* `profile range`: Split between command bytes 0x03 and 0x04. Same format as `profile range` in EEPROM.
* `random`: Enables random cycling mode when set to 1'b1, otherwise cycling is sequential.
* `attack`, `sustain`, and `release`: See [IR Command Fields: Attack, Sustain, and Release](#ir-command-fields-attack-sustain-and-release)


### IR Command: Display Single Color
Briefly display a single color.

The RGB values will be stored in CFG0 memory's static profile. The attack and release timers are always 0ms and 32ms, respectively. If `gsten=1'b1`, then the sustain time is set from [GST memory](#global-sustain-time-gst). Otherwise, sustain time is set at 384ms.

Flags: `type=2'b00`, `/rgb=1'b0`, (`onstrt=1'b1` and `gsten=1'b1`) or (`onstrt=1'b0` and `gsten=1'bX`)

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |                        green[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |                         red[7:2]                          |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |                         blue[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

> [!NOTE]
> Only the upper 6 bits of each RGB value are present in the command. The lower 2 bits are implicitly zero.


### IR Command: Display Single Color (Configurable Fields)
Briefly display a single color, but with additional configurable fields such as attack, sustain, release, and chance.

The RGB values along with the attack, sustain, and release timers will be stored in CFG0 memory.

Flags: `type=2'b00`, `/rgb=1'b0`, (`onstrt=1'b1` and `gsten=1'b1`) or (`onstrt=1'b0` and `gsten=1'bX`)

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |                        green[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |                         red[7:2]                          |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |                         blue[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    |           attack            |           chance            |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |           release           |           sustain           |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                     restrict group id                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Fields:
* `attack`, `sustain`, and `release`: See [IR Command Fields: Attack, Sustain, and Release](#ir-command-fields-attack-sustain-and-release)
* `chance`: See [IR Command Fields: Chance](#ir-command-fields-chance)
* `restrict group id`: See [IR Command Fields: Restrict Group ID](#ir-command-fields-restrict-group-id)

### IR Command: Color 1 Rapidly Followed by Color 2
Color 1 is briefly displayed for approximately 25ms with no attack or release timers (these intervals are not user-configurable), followed by Color 2 for a slightly longer period. The RGB values of Color 2 are saved in CFG0 memory.

The attack and release timers on Color 2 are always set at 32ms. If `gsten=1'b1`, then the sustain time for Color 2 is set from [GST memory](#global-sustain-time-gst). Otherwise, sustain time for Color 2 is set at 384ms.

Flags: `type=2'b01`, `/rgb=1'b0`, `onstrt=1'b0`, `gsten=1'bX`

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |                        green1[7:2]                        |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |                         red1[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |                        blue1[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    |                        green2[7:2]                        |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |                         red2[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                        blue2[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```


### IR Command: Set Color Profile
Set RGB color profile.

Flags: `type=2'b11`, `/rgb=1'b1`, `onstrt=1'b1`, `gsten=1'bX`

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |                        green[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |                         red[7:2]                          |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |                         blue[7:2]                         |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    | skpdisp |  /save  |              profile id               |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |    0    |    0    |    0    |    0    |    0    |    0    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                     restrict group id                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Fields:
* `skpdisp` When set to 1, the PixMob will be "silent" and not display the color when the command is received. Otherwise, when set to 0, the color will be briefly displayed at the time the command is received.
* `/save`: When set to 1'b0, the color profile is saved to the EEPROM at the specified profile id. Otherwise, when set to 1'b1, the color profile is stored elsewhere in memory (exact use case still under investigation).
* `profile id`: The index of the profile within EEPROM to save to. Valid values are 0 to 15.
* `restrict group id`: See [IR Command Fields: Restrict Group ID](#ir-command-fields-restrict-group-id)


### IR Command: Change Group
Change the PixMob device's group by setting the `group sel` field in EEPROM.

Flags: `type=2'b11`, `/rgb=1'b1`, `onstrt=1'b1`, `gsten=1'bX`

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |     red[5:4]      |               green[7:4]              |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |               blue[7:4]               |     red[7:6]      |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |    X    |    X    |    X    |          group sel          |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    | skpdisp |    X    |    X    |    X    |    X    |    X    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |    0    |    0    |    0    |    0    |    0    |    1    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                     restrict group id                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Fields:
* `green`, `red`, and `blue` is a compacted 12-bit RGB that is copied into CFG0 memory.
* `group sel`: Changes the group id selection. The new `group sel` value is written to EEPROM and the corresponding new `group id` is read from EEPROM at the selected offset.
* `skpdisp` When set to 1, the PixMob will be "silent" and not display the color when the command is received. Otherwise, when set to 0, the color will be briefly displayed at the time the command is received.
* `restrict group id`: See [IR Command Fields: Restrict Group ID](#ir-command-fields-restrict-group-id)

> [!NOTE]
> The group id matching takes place prior to command execution. In other words, the `group id` field will match against the old group id, not the new one the PixMob is being updated to.


### IR Command: Set Group ID
Set one of eight group ids in the EEPROM.

Flags: `type=2'b11`, `/rgb=1'b1`, `onstrt=1'b1`, `gsten=1'bX`

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |     red[5:4]      |               green[7:4]              |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |               blue[7:4]               |     red[7:6]      |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |    X    |    X    |    X    |          group sel          |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    | skpdisp |                  new group id                   |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |    0    |    0    |    0    |    0    |    1    |    0    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                     restrict group id                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

Fields:
* `green`, `red`, and `blue` is a compacted 12-bit RGB that is copied into CFG0 memory.
* `group sel`: Selects which of the eight `group sel * id`s to change in the EEPROM.
* `new group id`: The new 5-bit group ID to write into the EEPROM.
* `skpdisp` When set to 1, the PixMob will be "silent" and not display the color when the command is received. Otherwise, when set to 0, the color will be briefly displayed at the time the command is received.
* `restrict group id`: See [IR Command Fields: Restrict Group ID](#ir-command-fields-restrict-group-id)

> [!WARNING]
> The PixMob does not update the cached group id in memory when executing this command. It is only written to the EEPROM. If the `group sel` field is changing the current group id, the PixMob still retains the old group id until either it is rebooted (and data re-read from EEPROM), or it receives a [change group command](#ir-command-change-group).

Special Cases:
* If `new group id=0`, the command will be discarded and nothing changed in EEPROM.


### IR Command: Set GST
Set the [Global Sustain Time (GST)](#global-sustain-time-gst) memory.

Flags: `type=2'b11`, `/rgb=1'b1`, `onstrt=1'b1`, `gsten=1'bX`

```
          7         6         5         4         3         2         1         0     
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x03 |    0    |    0    |    X    |    X    |    X    |    X    |    X    |    X    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x04 |    0    |    0    |    X    |    X    |    X    |           gst key           |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x05 |    0    |    0    |    X    |    X    |    X    |    X    |    X    |    X    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x06 |    0    |    0    |    X    |    X    |    X    |    X    |    X    |    X    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x07 |    0    |    0    |    0    |    0    |    1    |    0    |    0    |    1    |
     +---------+---------+---------+---------+---------+---------+---------+---------+
0x08 |    0    |    0    |                     restrict group id                     |
     +---------+---------+---------+---------+---------+---------+---------+---------+
```

`gst key` is a 3-bit field used to index into the following table:

| Value  | Time (milliseconds) |
| :----: | :-----------------: |
| 3'b000 |           64        |
| 3'b001 |          112        |
| 3'b010 |          160        |
| 3'b011 |          208        |
| 3'b100 |          480        |
| 3'b101 |          960        |
| 3'b110 |        2,400        |
| 3'b111 |        3,840        |


### IR Command Fields: Attack, Sustain, and Release
The terms attack, sustain, and release are used to describe the LED fade-in time, hold time, and fade-out time, respectively. With some limited exceptions, the mapping of the 3-bit values are:

| Value  | Time (milliseconds) |
| :----: | :-----------------: |
| 3'b000 |            0        |
| 3'b001 |           32        |
| 3'b010 |           96        |
| 3'b011 |          192        |
| 3'b100 |          480        |
| 3'b101 |          960        |
| 3'b110 |        2,400        |
| 3'b111 |        3,840        |

Special Cases:
* If `sustain=3'b111` and `release!=3'b000`, then the sustain timer is set from [GST memory](#global-sustain-time-gst).
* If `release=3'b000`, then some type of multiplier is applied to the sustain time. Exact details are still under investigation.


### IR Command Fields: Chance
Chance refers to the probability that the command will be executed. The mapping of the 3-bit values are:

| Value | Probability (%) |
| :---: | :-------------: |
| 3'b000 |       100       |
| 3'b001 |        88       |
| 3'b010 |        67       |
| 3'b011 |        50       |
| 3'b100 |        32       |
| 3'b101 |        16       |
| 3'b110 |        10       |
| 3'b111 |         4       |


### IR Command Fields: Restrict Group ID
IR commands with a group id will restrict execution of the command only to PixMob devices in the matching group (determined by data in the EEPROM).

Special Cases:
* If `group id=5'd0`, then `group id` is set to 5'd1 before checking for a matching group.


### IR Command Encoding
A lookup table is used to transform a 6-bit command byte into an 8-bit encoded command byte.

The lower 6 bits of each command byte are used to index into the following table:

```c
0x21, 0x32, 0x54, 0x65, 0xa9, 0x9a, 0x6d, 0x29, /* 0x00 - 0x07 */
0x56, 0x92, 0xa1, 0xb4, 0xb2, 0x84, 0x66, 0x2a, /* 0x08 - 0x0F */
0x4c, 0x6a, 0xa6, 0x95, 0x62, 0x51, 0x42, 0x24, /* 0x10 - 0x17 */
0x35, 0x46, 0x8a, 0xac, 0x8c, 0x6c, 0x2c, 0x4a, /* 0x18 - 0x1F */
0x59, 0x86, 0xa4, 0xa2, 0x91, 0x64, 0x55, 0x44, /* 0x20 - 0x27 */
0x22, 0x31, 0xb1, 0x52, 0x85, 0x96, 0xa5, 0x69, /* 0x28 - 0x2F */
0x5a, 0x2d, 0x4d, 0x89, 0x45, 0x34, 0x61, 0x25, /* 0x30 - 0x37 */
0x36, 0xad, 0x94, 0xaa, 0x8d, 0x49, 0x99, 0x26, /* 0x38 - 0x3F */
```

The primary purpose of this encoding is likely to minimize the number of consecutive 1's and 0's both within and across neighboring command bytes being sent by the IR transmitter. Since the IR transmission is based on ~700 microsecond time-sliced segments, too many consecutive bits of the same value may be interpreted as loss of signal or interference.

> [!NOTE]
> The magic constant at command offset 0x00 is NOT encoded!


### Checksum Calculation
A partial checksum is first calculated by summing the values of the **encoded** command bytes, starting at offset 0x02 until the end of the command.

The upper 6 bits of the partial checksum are then used to index into the [IR command encoding table](#ir-command-encoding), with the resulting 8-bit value being used as the final checksum.

