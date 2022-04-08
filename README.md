AzulSCSI Firmware
=================

Harddrive image files
---------------------
AzulSCSI uses the same type of (raw) hard drive image file as in [BlueSCSI instructions](https://github.com/erichelgeson/BlueSCSI/wiki/Usage).

Examples of valid filenames:
* `HD5.hda` or `HD5.img`: hard drive with SCSI ID 5
* `HD20_512.hda`: hard drive with SCSI ID 2, LUN 0, block size 512
* `CD3.iso`: CD drive with SCSI ID 3

Log files and error indications
-------------------------------
Log messages are stored in `azullog.txt`, which is cleared on every boot.
Normally only basic initialization information is stored, but switching the `DBG` DIP switch on will cause every SCSI command to be logged, once the board is power cycled.

The indicator LED will normally report disk access.
It also reports following status conditions:

- 1 fast blink on boot: Image file loaded successfully
- 3 fast blinks: No images found on SD card
- 5 fast blinks: SD card not detected
- Continuous morse pattern: firmware crashed, morse code indicates crash location

In crashes the firmware will also attempt to save information into `azulerr.txt`.

Configuration file
------------------
Optional configuration can be stored in `azulscsi.ini`.
If image file is found but configuration is missing, a default configuration is used.

Example config file is available here: [azulscsi.ini](azulscsi.ini).

Performance
-----------
With verbose log messages disabled, expected SCSI performance is 2.4 MB/s read and 2.1 MB/s write.
Slow SD card or fragmented filesystem can slow down access.

Seek performance is best if image files are contiguous.
For ExFAT filesystem this relies on a file flag set by PC.
Current versions of exfat-fuse on Linux have an [issue](https://github.com/relan/exfat/pull/101) that causes the files not to be marked contiguous even when they are.
This is indicated by message `WARNING: file HD00_512.hda is not contiguous. This will increase read latency.` in the log.

Hotplugging
-----------
The firmware supports hot-plug removal and reinsertion of SD card.
The status led will blink continuously when card is not present, then blink once when card is reinserted successfully.

It will depend on the host system whether it gets confused by hotplugging.
Any IO requests issued when card is removed will be timeouted.

Programming & bootloader
------------------------
There is a bootloader that loads new firmware from SD card on boot.
The firmware file must be e.g. `AzulSCSI.bin` or `AzulSCSIv1_0_2022-xxxxx.bin`.
Firmware update takes about 1 second, during which the LED will flash rapidly.

When successful, the bootloader removes the update file and continues to main firmware.
On failure, `azulerr.txt` is written on the SD card.

Alternatively, the board can be programmed using USB connection in DFU mode by setting DIP switch 4.
The necessary programmer utility for Windows can be downloaded from [GD32 website](http://www.gd32mcu.com/en/download?kw=dfu&lan=en). On Linux and MacOS, the standard 'dfu-util' can be used. It can be installed via your package manager under Linux. On MacOS, it is available through MacPorts and Brew as a package

DIP switches
------------
The DIP switch settings are as follows:

- DEBUG: Enable verbose debug log (saved to `azullog.txt`)
- TERM: Enable SCSI termination
- BOOT: Enable built-in USB bootloader, this DIP switch MUST remain off during normal operation.
- SW1: Enables/disables Macintosh/Apple specific mode-pages and device strings, which eases disk initialization when performing fresh installs on legacy Macintosh computers.

Project structure
-----------------
- **src/AzulSCSI.cpp**: Main portable SCSI implementation.
- **src/AzulSCSI_disk.cpp**: Interface between SCSI2SD code and SD card reading.
- **src/AzulSCSI_log.cpp**: Simple logging functionality, uses memory buffering.
- **src/AzulSCSI_config.h**: Some compile-time options, usually no need to change.
- **lib/AzulSCSI_platform_GD32F205**: Platform-specific code for GD32F205.
- **lib/SCSI2SD**: SCSI2SD V6 code, used for SCSI command implementations.
- **lib/minIni**: Ini config file access library
- **lib/SdFat_NoArduino**: Modified version of [SdFat](https://github.com/greiman/SdFat) library for use without Arduino core.
- **utils/run_gdb.sh**: Helper script for debugging with st-link adapter. Displays SWO log directly in console.

To port the code to a new platform, see README in [lib/AzulSCSI_platform_template](lib/AzulSCSI_platform_template) folder.

Building
--------
This codebase uses [PlatformIO](https://platformio.org/).
To build run the command:

    pio run


Origins and License
-------------------

This firmware is derived from two sources, both under GPL 3 license:

* [SCSI2SD V6](http://www.codesrc.com/mediawiki/index.php/SCSI2SD)
* [BlueSCSI](https://github.com/erichelgeson/BlueSCSI), which in turn is derived from [ArdSCSIno-stm32](https://github.com/ztto/ArdSCSino-stm32).

Main program structure:

* SCSI command implementations are from SCSI2SD.
* SCSI physical layer code is mostly custom, with some inspiration from BlueSCSI.
* Image file access is derived from BlueSCSI.

Major changes from BlueSCSI and SCSI2SD include:

* Separation of platform-specific functionality to separate directory to ease porting.
* Ported to GD32F205.
* Removal of Arduino core dependency, as it was not currently available for GD32F205.
* Buffered log functions.
* Simultaneous transfer between SD card and SCSI for improved performance.
