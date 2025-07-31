# esPod

## Prior Art

- [Guy Dupont's sPot](https://www.youtube.com/watch?v=ZxdhG1OhVng)
    - Honestly incredibly janky and poorly designed
        - A mess of wires and random breakout boards stuffed haphhazardly into the case
        - Lock switch just pulls power to the board
        - Analog composite signal to drive a crappy looking display
        - Cheap USB DAC
        - Cheap vibration motor instead of clicker
        - Smaller than stock battery, and presumably way worse battery life
    - Derivatives:
        - [RSFlighttronics' SpotifyPod](https://rsflightronics.com/spotifypod)
            - Only marginally better
                - Still a mess of hand soldered wires and breakouts
                - Digital display
                - Display is clearly poorly aligned with cutout
                - 3D printed bracket to house boards
- [My own piPod](https://github.com/Gigahawk/pipod_hw)
    - The general concept of an iPod board replacement has been rattling in my head for a really long time, this was my first (unfinished/untested attempt)
    - Closely matches the original iPod board shape
    - Uses a STP240320_0200B display with this [board to board adapter](https://github.com/Gigahawk/STP240320_0200B_b2b) and a 3D printed shell to mount securely into housing with good alignment
    - DAC choices limited due to RPi's inability to generate I2S MCLK
    - Uses a coprocessor to translate clickwheel SPI to I2C and handle standby
        - In theory it might have been possible to elimitate this with the BTN1 method described below, but some work would have been needed to get the Pi to act as a SPI slave on SPI1/2 (SPI1 is dedicated to driving the display)
    - Robust USB-C connector
    - No boost converter for main RPi
        - The RPi is actually capable of running from a wide variety of input voltages, seemingly all the way down to near 3.3V https://hackaday.io/project/19035-zerophone-a-raspberry-pi-smartphone/log/69596-a-guide-on-powering-pi-zero-directly-from-liion-batteries
    - Requires a second compute board to actually house the RPi
        - RPi would need connectors desoldered to actually fit in the case.
    - Used a stock battery probably would have had poor battery life
    - Use of a SBC was driven laregly out of desire for supporting Syncthing
- [Tangara](https://cooltech.zone/tangara/)
    - iPod inspired, doesn't use a real iPod enclosure
    - ESP32 based, apparently this is enough to handle MP3 and FLAC decoding
    - Uses a full sized SD card for some reason
    - Needs a coprocessor for power management
    - Supposedly 20+ hours of active listening time with a 2200mAh battery
    - What appears to be a fairly well thought out DAC/amp chain: https://cooltech.zone/tangara/blog/2024-02-14-audio-quality/


## HW Design Goals/Requirements


### User Facing

Some features may be more complicated to enable in firmware, but the hardware should at least support most of the following:

1. Good battery life (TBD on what that looks like)
    - With no harddrive it may be possible to handle two standard batteries in parallel.
        - Ideally board should have a warning that cells must be balanced when connecting
    - Going by Tangaras numbers it should be possible to achieve a similar 20ish hour runtime with two standard batteries in parallel
2. As much "original" iPod functionality as possible
    - Headphone output (good DAC, TBD on what that might be)
    - Lock switch
    - Expansion port
    - TV out support?
        - Might be better to support audio input idk
    - USB data transfer
    - Screen
    - Clickwheel
    - Clickwheel clicker
    - Original battery support
3. USB-C
    - ~~Host mode support for USB DACs, maybe USB-C analog audio out~~ Original ESP32 does not support USB
4. WiFi file sync
    - Using a MCU means no syncing via Syncthing or even easy connectivity to a private cloud with Tailscale etc.
    - Closest we can probably do is an rsync compatible client and an IP/URL to try to pull updates from
5. Bluetooth audio
6. Passthrough mode
    - Sources:
        - Bluetooth (iPod acts as Bluetooth speaker)
        - ~~USB (iPod acts as audio device)~~ Not supported on base ESP32
        - ~~USB (iPod acts as analog audio device)~~ Not supported on base ESP32
        - Headphone jack (ADC connected to headphone jack?)
        - WiFi (internet radio/streaming?)
        - Local storage
    - Sinks:
        - Bluetooth (iPod streams to bluetooth speaker)
        - ~~USB (iPod sends audio to USB audio device)~~ Not supported on base ESP32
        - Headphone jack
7. SD bulk storage
    - May be housed internally


### Architecture

- Main CPU: ESP32, ~~probably S2/S3 for native USB support~~ must use a regular ESP32 for BT classic (audio support)
    - ~~OMGS3 could be used to simplify layout, but this will prevent doing complete power off for deep sleep, maybe just use it to inspire component selection~~
    - Since we will probably be forking the Tangara firmware as a start might as well follow their design as much as we can
- No dedicated coprocessor for standby, main CPU is completely off for deep sleep
    - We can use the clickwheel processor BTN1 to wake up/reenable power to main CPU, CPU can latch power on until it's time to go back to standby
- Interchangeable internal DAC/Amp?
    - Presumably we can put the DAC on a stamp with all I2S signals and analog outputs routed to the edge
- Probably a chip antenna, maybe near the USB port?
    - The metal rear of the case is probably not that great for reception, but existing projects don't seem to have too much trouble with it. That said the built in trace antenna on the RPi Zero is apparently quite fancy and uses some licensed tech that we can't use.
    - Will probably fit on the underside of the board under the screen, that way it will be pointing towards the plastic portion of the case? hopefully won't interfere with audio path, should be ok since that section is past the amp.
- USB data transfer
    - With no native USB support, may have to mux the SD card with a USB to SD adapter or something, possibly also add a hub for a built in programmer?
    - USB Hubs
        - CoreChips [SL2.1A](https://lcsc.com/product-detail/USB-HUB-Controllers_CoreChips-SL2-1A_C192893.html)/[SL2.1S](https://lcsc.com/product-detail/USB-HUB-Controllers_CoreChips-SL2-1s_C2684433.html) are super cheep from LCSC, chinese only datasheet but seems simple enough to implement
        - [USB2422](https://www.digikey.ca/en/products/detail/microchip-technology/USB2422-MJ/4080182) is somewhat pricy and can't be direct powered from 5V
    - SD reader (Turns out adapter chips are somewhat hard to come by)
        - There's the [MAX14502](https://www.digikey.com/en/products/detail/analog-devices-inc-maxim-integrated/MAX14502AETL/2061945) which integrates a built in bypass mode which would eliminate the need for a hub but is either not available or super expensive
        - The [GL823K](https://lcsc.com/product-detail/USB-Converters_GENESYS-GL823K-HCY04_C284879.html?s_z=h_GL823) is available only from LCSC
        - The [USB2241I](https://www.digikey.ca/en/products/detail/microchip-technology/USB2241I-AEZG-06/3873145?s=N4IgTCBcDaIKoGUBCYwBYCMBJEBdAvkA) is a little pricy and can't be direct powered from 5V
    - Serial adapter
        - [CH9102](https://lcsc.com/product-detail/USB-Converters_WCH-CH9102F_C2858418.html) somewhat pricy but seems to support 5V power
        - [CP2105](https://www.digikey.com/en/products/detail/silicon-labs/CP2105-F01-GM/2486179) quite expensive and seems overkill
        - [FT230X](https://www.mouser.ca/ProductDetail/FTDI/FT230XS-R?qs=Gp1Yz1mis3XyCLeYOseSng%3D%3D) Somewhat pricy but also comes in convenient SSOP and great docs


### Misc

- No permanent case mods
