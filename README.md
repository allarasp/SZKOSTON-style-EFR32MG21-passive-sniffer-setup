# SZKOSTON-style EFR32MG21 Passive Sniffer Setup

![SZKOSTON-style EFR32MG21 USB dongle](images/szkoston-efr32mg21-dongle.jpg)

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

The tested custom sniffer firmware is included at:

```text
firmware/source/SnifferNCP.gbl
```

Recommended layout:

```text
README.md
images/
  szkoston-efr32mg21-dongle.jpg
firmware/
  README.md
  source/
    SnifferNCP.gbl
```

## 1. Install Node.js

`ember-zli` is a Node.js command-line application, so install Node.js first. On Windows, the easiest method is to install the current Node.js LTS release from the official Node.js website:

https://nodejs.org/en/download

The Node.js installer also installs `npm`, which is used to install `ember-zli`.

After installation, close and reopen PowerShell and verify both commands:

```powershell
node --version
npm --version
```

Both commands should print a version number.

If Windows Package Manager (`winget`) is available, Node.js LTS can alternatively be installed from PowerShell with:

```powershell
winget install OpenJS.NodeJS.LTS
```

Close and reopen PowerShell after installation, then verify:

```powershell
node --version
npm --version
```

## 2. Install ember-zli

Install `ember-zli` globally with npm:

```powershell
npm install -g ember-zli
```

Verify the installation:

```powershell
ember-zli --version
```

You can also display the available commands with:

```powershell
ember-zli --help
```

The command used for passive packet capture is:

```powershell
ember-zli sniff
```

If PowerShell reports that `ember-zli` is not recognized after installation, close and reopen the terminal first so that the updated npm PATH is loaded.

Official ember-zli project:

https://github.com/Nerivec/ember-zli

## 3. Install Tera Term

Download and install **Tera Term** for Windows.

Tera Term is used to open the CH340 COM port, access the Gecko bootloader, and transfer the `.gbl` firmware file using XMODEM.

After installation, connect the USB dongle to the PC. Open **Device Manager -> Ports (COM & LPT)**. The tested adapter appeared as `USB-SERIAL CH340 (COM4)`. Your COM number may be different.

## 4. Put the dongle into Gecko bootloader mode

The tested SZKOSTON-style board has two buttons/signals: `BOOT` and `nRST` / reset.

Use this exact sequence:

1. **Press and hold `BOOT`.**
2. While still holding `BOOT`, **press and release `nRST`.**
3. **Release `BOOT`.**

```text
Hold BOOT
  -> press nRST
  -> release nRST
  -> release BOOT
```

## 5. Open the serial port in Tera Term

Start Tera Term, select **Serial**, choose the CH340 COM port, and use `115200`, 8 data bits, no parity, 1 stop bit. For the bootloader transfer hardware RTS/CTS is not required.

If the bootloader is active, Tera Term should show the Gecko bootloader menu or prompt. If normal NCP/Zigbee traffic appears instead, repeat the BOOT + nRST sequence.

## 6. Flash the custom sniffer firmware with XMODEM

Use the firmware included in this repository:

```text
firmware/source/SnifferNCP.gbl
```

Once the Gecko bootloader is visible, choose the bootloader option that starts an XMODEM firmware upload. Then in Tera Term use:

```text
File -> Transfer -> XMODEM -> Send
```

Select `firmware/source/SnifferNCP.gbl`, start the transfer, and wait until it is completely finished. Do not unplug the dongle during upload. If it does not reboot automatically, press `nRST` once.

## 7. Expected firmware configuration

The custom firmware used for this setup was built in **Simplicity Studio 6** from the Silicon Labs `Zigbee - NCP UART HW` example for `EFR32MG21A020F768IM32`, with `Zigbee -> Utility -> Manufacturing Library` enabled.

Working UART configuration:

```text
USART0
TX: PB01
RX: PB00
baud: 115200
software flow control
RTS/CTS: disabled
```

The tested NCP reported `9.1.1 [GA]`, EZSP protocol version `19`.

## 8. Start a passive capture

Make sure Tera Term and any other application using the dongle's COM port are closed.

Open PowerShell and run:

```powershell
ember-zli sniff
```

For the tested dongle select:

```text
Connection: Serial
Baud rate: 115200
Port: your CH340 COM port
Flow control: Software
RTS/CTS: false
Output: PCAP file
```

Select the Zigbee channel you want to monitor. For the Schneider PowerTag commissioning capture we used:

```text
Channel: 11
TX power: 5
```

A successful startup should report the NCP/EZSP version, start MFGLIB, and end with:

```text
Sniffing started.
```

Leave the command running while performing the Zigbee/Green Power activity you want to capture. Stop the capture with `Ctrl+C` when finished.

## 9. Restoring the original coordinator firmware

Keep a known-good recovery image. For the tested stick, the recovery image used was `ncp-uart-sw_7.4.3.0_115200.gbl`.

Restore it with the same BOOT/nRST sequence and Tera Term XMODEM procedure.

## Troubleshooting

If Tera Term shows nothing, verify the COM port and 115200 baud and make sure no other program owns the serial port. If normal firmware starts, repeat the BOOT/nRST sequence. If XMODEM does not start, first make sure the Gecko bootloader has entered XMODEM receive mode. Close Tera Term before starting `ember-zli`. If no packets are captured, verify the radio channel. If MFGLIB does not start, verify that Manufacturing Library was included in the firmware build.

If `node`, `npm`, or `ember-zli` is reported as an unknown command, close and reopen PowerShell after installation and check the commands again.

## Related project

This sniffer setup was created while investigating Schneider PowerTag Zigbee Green Power commissioning. See the `Schneider-PowerTag-Zigbee` repository for commissioning analysis and example capture data.

## Disclaimer

This guide documents development and interoperability work performed on owned hardware. Flashing third-party firmware can make a dongle temporarily unusable if the wrong image or hardware configuration is used. Always keep a recovery image and verify the target MCU before flashing.