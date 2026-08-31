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

The custom firmware file should be placed in:

```text
firmware/PowerTagSnifferNCP.gbl
```

This repository intentionally does not include the firmware binary yet. Add the tested `.gbl` file to the `firmware/` directory when it is ready.

Recommended layout:

```text
README.md
images/
  szkoston-efr32mg21-dongle.jpg
firmware/
  PowerTagSnifferNCP.gbl
  README.md
```

## 1. Install Tera Term

Download and install **Tera Term** for Windows.

Tera Term is used to open the CH340 COM port, access the Gecko bootloader, and transfer the `.gbl` firmware file using XMODEM.

After installation, connect the USB dongle to the PC. Open **Device Manager -> Ports (COM & LPT)**. The tested adapter appeared as `USB-SERIAL CH340 (COM4)`. Your COM number may be different.

## 2. Put the dongle into Gecko bootloader mode

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

## 3. Open the serial port in Tera Term

Start Tera Term, select **Serial**, choose the CH340 COM port, and use `115200`, 8 data bits, no parity, 1 stop bit. For the bootloader transfer hardware RTS/CTS is not required.

If the bootloader is active, Tera Term should show the Gecko bootloader menu or prompt. If normal NCP/Zigbee traffic appears instead, repeat the BOOT + nRST sequence.

## 4. Flash the custom sniffer firmware with XMODEM

The firmware should be stored at:

```text
firmware/PowerTagSnifferNCP.gbl
```

Once the Gecko bootloader is visible, choose the bootloader option that starts an XMODEM firmware upload. Then in Tera Term use:

```text
File -> Transfer -> XMODEM -> Send
```

Select `firmware/PowerTagSnifferNCP.gbl`, start the transfer, and wait until it is completely finished. Do not unplug the dongle during upload. If it does not reboot automatically, press `nRST` once.

## 5. Expected firmware configuration

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

## 6. Start a passive capture

Open PowerShell and run:

```powershell
ember-zli sniff
```

Use Serial, 115200 baud, your CH340 COM port, Software flow control, RTS/CTS false, and PCAP output. For the tested PowerTag capture we used channel 11 and radio TX power 5.

A successful start should end with:

```text
Sniffing started.
```

## 7. Restoring the original coordinator firmware

Keep a known-good recovery image. For the tested stick, the recovery image used was `ncp-uart-sw_7.4.3.0_115200.gbl`.

Restore it with the same BOOT/nRST sequence and Tera Term XMODEM procedure.

## Troubleshooting

If Tera Term shows nothing, verify the COM port and 115200 baud and make sure no other program owns the serial port. If normal firmware starts, repeat the BOOT/nRST sequence. If XMODEM does not start, first make sure the Gecko bootloader has entered XMODEM receive mode. Close Tera Term before starting `ember-zli`. If no packets are captured, verify the radio channel. If MFGLIB does not start, verify that Manufacturing Library was included in the firmware build.

## Related project

This sniffer setup was created while investigating Schneider PowerTag Zigbee Green Power commissioning. See the `Schneider-PowerTag-Zigbee` repository for commissioning analysis and example capture data.

## Disclaimer

This guide documents development and interoperability work performed on owned hardware. Flashing third-party firmware can make a dongle temporarily unusable if the wrong image or hardware configuration is used. Always keep a recovery image and verify the target MCU before flashing.