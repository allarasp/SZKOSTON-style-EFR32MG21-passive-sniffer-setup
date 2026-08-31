# SZKOSTON-style EFR32MG21 Passive Sniffer Setup

This repository documents how to convert a SZKOSTON-style EFR32MG21 Zigbee USB dongle into a passive IEEE 802.15.4 / Zigbee sniffer using a custom Silicon Labs NCP firmware with MFGLIB support.

The guide is based on a tested dongle using:

- MCU: `EFR32MG21A020F768IM32`
- USB-UART bridge: `CH340`
- UART baud rate: `115200`
- TX: `PB01`
- RX: `PB00`
- software flow control (`XON/XOFF`)
- no RTS/CTS

The intended use is passive radio capture with `ember-zli sniff`, producing PCAP files for later analysis.

> **Important:** Flashing custom firmware replaces the original coordinator firmware on the dongle. Keep a recovery image before you start.

## Repository layout

The custom firmware file should be placed in:

```text
firmware/PowerTagSnifferNCP.gbl
```

This repository intentionally does not include the firmware binary yet. Add the tested `.gbl` file to the `firmware/` directory when it is ready.

Recommended layout:

```text
README.md
firmware/
  PowerTagSnifferNCP.gbl
  README.md
```

## 1. Install Tera Term

Download and install **Tera Term** for Windows.

Tera Term is used to:

1. open the CH340 COM port,
2. access the Gecko bootloader,
3. transfer the `.gbl` firmware file using XMODEM.

After installation, connect the USB dongle to the PC.

Open **Device Manager** and check the COM port under:

```text
Ports (COM & LPT)
```

The tested adapter appeared as:

```text
USB-SERIAL CH340 (COM4)
```

Your COM number may be different.

## 2. Put the dongle into Gecko bootloader mode

The tested SZKOSTON-style board has two buttons/signals:

- `BOOT`
- `nRST` / reset

Use this exact sequence:

1. **Press and hold `BOOT`.**
2. While still holding `BOOT`, **press and release `nRST`**.
3. **Release `BOOT`.**

In short:

```text
Hold BOOT
  -> press nRST
  -> release nRST
  -> release BOOT
```

The dongle should now be in the Silicon Labs Gecko bootloader instead of starting the normal Zigbee NCP firmware.

## 3. Open the serial port in Tera Term

Start **Tera Term**.

In the connection dialog:

1. Select **Serial**.
2. Select the CH340 COM port, for example `COM4`.
3. Click **OK**.

Then open:

```text
Setup -> Serial port
```

Use:

```text
Speed:       115200
Data:        8 bit
Parity:      none
Stop bits:   1 bit
Flow control: none
```

For the bootloader transfer, hardware RTS/CTS is not required.

If the bootloader is active, Tera Term should show the Gecko bootloader menu or bootloader prompt.

If you see normal NCP/Zigbee traffic instead, repeat the BOOT + nRST sequence.

## 4. Flash the custom sniffer firmware with XMODEM

The firmware file for this project should be stored here:

```text
firmware/PowerTagSnifferNCP.gbl
```

Once the Gecko bootloader is visible in Tera Term, choose the bootloader option that starts an **XMODEM firmware upload**.

Then in Tera Term open:

```text
File -> Transfer -> XMODEM -> Send
```

Select:

```text
firmware/PowerTagSnifferNCP.gbl
```

Start the transfer.

Wait until the transfer is completely finished. Do not unplug the dongle during the upload.

After a successful upload, the bootloader should validate the image and boot the new firmware.

If it does not reboot automatically, press `nRST` once.

## 5. Expected firmware configuration

The custom firmware used for this setup was built in **Simplicity Studio 6** from the Silicon Labs Zigbee NCP UART example.

Target MCU:

```text
EFR32MG21A020F768IM32
```

Base example:

```text
Zigbee - NCP UART HW
```

Required component:

```text
Zigbee -> Utility -> Manufacturing Library
```

MFGLIB is required because `ember-zli sniff` uses the manufacturing-library receive path for passive radio capture.

The working UART configuration is:

```text
USART0
TX: PB01
RX: PB00
baud: 115200
software flow control
RTS/CTS: disabled
```

A generated configuration from the tested build contained:

```c
#define SL_IOSTREAM_USART_VCOM_FLOW_CONTROL_TYPE uartFlowControlSoftware
```

The tested NCP reported:

```text
NCP version: 9.1.1 [GA]
EZSP protocol version: 19
```

## 6. Install ember-zli

The passive capture is started with `ember-zli`.

Make sure Node.js and the required package/tool are installed so that this command works from PowerShell:

```powershell
ember-zli sniff
```

If the command is not found, install the `ember-zli` tool first and then reopen PowerShell.

## 7. Start a passive capture

Open PowerShell and run:

```powershell
ember-zli sniff
```

Use these settings when prompted:

```text
Transport: Serial
Baud rate: 115200
COM port: your CH340 COM port
Flow control: Software
RTS/CTS: false
Output: PCAP
```

For the tested PowerTag capture we used:

```text
COM4
channel 11
radio TX power 5
```

The important part is that the sniffer channel must match the Zigbee / 802.15.4 channel used by the devices you want to observe.

When everything is working, the tool should report something similar to:

```text
NCP EZSP protocol version (19) matches Host.
NCP version: 9.1.1 [GA]
Sniffing started.
```

At this point the dongle is acting as a passive sniffer.

## 8. Capture topology

The sniffer does not need to be the Zigbee coordinator.

For example:

```text
Device A  <------ Zigbee / Green Power ------>  Gateway
                      |
                      |
                      +---- EFR32MG21 sniffer -> PCAP
```

The dongle only listens on the selected radio channel and writes received frames to the capture file.

## 9. Restoring the original coordinator firmware

Keep a known-good recovery image before flashing the sniffer firmware.

For the tested EFR32MG21 stick, a compatible original NCP image used during recovery was:

```text
ncp-uart-sw_7.4.3.0_115200.gbl
```

To restore it, use the same procedure:

1. Hold `BOOT`.
2. Press and release `nRST`.
3. Release `BOOT`.
4. Open the COM port in Tera Term.
5. Start XMODEM receive in the Gecko bootloader.
6. Use:

```text
File -> Transfer -> XMODEM -> Send
```

7. Select the recovery `.gbl` image.
8. Wait for the transfer to finish.
9. Reset the dongle.

## 10. Troubleshooting

### Tera Term shows nothing

Check:

- correct COM port,
- 115200 baud,
- the USB dongle is actually connected,
- no other application has the COM port open.

Then repeat the bootloader sequence:

```text
hold BOOT -> tap nRST -> release BOOT
```

### The normal NCP firmware starts instead of the bootloader

The BOOT button was probably not held at the moment reset was released. Repeat the sequence more carefully.

### XMODEM transfer does not start

First make sure the Gecko bootloader itself has entered its XMODEM receive mode. Only then choose:

```text
File -> Transfer -> XMODEM -> Send
```

in Tera Term.

### ember-zli cannot open the COM port

Close Tera Term completely before starting `ember-zli`. Only one program can normally own the serial port at a time.

### No packets are captured

The most common cause is the wrong radio channel. Set the sniffer to the same channel as the target Zigbee network.

### ember-zli connects but MFGLIB does not start

Verify that the custom firmware was built with the Silicon Labs **Manufacturing Library** component enabled.

## 11. Tested hardware notes

The specific tested adapter was a SZKOSTON-style Zigbee 3.0 USB dongle based on:

```text
EFR32MG21A020F768IM32
CH340 USB-UART
PCB antenna
```

Its original firmware was an Ember NCP at 115200 baud. The board behaved similarly to other EFR32MG21 coordinator sticks, but the UART bridge and exact pin mapping matter when building custom firmware.

Do not assume this firmware will work unchanged on every EFR32MG21 USB dongle. Confirm the MCU, UART pins and USB-UART configuration first.

## 12. Related project

This sniffer setup was created while investigating Schneider PowerTag Zigbee Green Power commissioning.

Related repository:

https://github.com/allarasp/Schneider-PowerTag-Zigbee

That repository contains the captured commissioning analysis and example PCAP/log data produced with this sniffer setup.

## Disclaimer

This guide documents development and interoperability work performed on owned hardware. Flashing third-party firmware can make a dongle temporarily unusable if the wrong image or hardware configuration is used. Always keep a recovery image and verify the target MCU before flashing.