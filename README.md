# SZKOSTON-style EFR32MG21 Passive Sniffer Setup

![SZKOSTON-style EFR32MG21 USB dongle](images/szkoston-efr32mg21-dongle.jpg)

This repository documents how to convert a SZKOSTON-style EFR32MG21 Zigbee USB dongle into a passive IEEE 802.15.4 / Zigbee sniffer using a custom Silicon Labs NCP firmware with MFGLIB support, and how to restore the dongle back to coordinator firmware afterwards.

The guide is based on a tested dongle using:

- MCU: `EFR32MG21A020F768IM32`
- USB-UART bridge: `CH340`
- UART baud rate: `115200`
- TX: `PB01`
- RX: `PB00`
- software flow control (`XON/XOFF`)
- no RTS/CTS

The intended use is passive radio capture with `ember-zli sniff`, producing PCAP files for later analysis.

> **Important:** Flashing custom firmware replaces the coordinator firmware on the dongle. A tested recovery firmware is included in this repository so the dongle can be restored afterwards.

## Repository layout

Both required firmware images are included:

```text
README.md
images/
  szkoston-efr32mg21-dongle.jpg
firmware/
  README.md
  source/
    SnifferNCP.gbl
    ncp-uart-sw_7.4.3.0_115200.gbl
```

Firmware purpose:

| File | Purpose |
| --- | --- |
| `firmware/source/SnifferNCP.gbl` | Custom MFGLIB NCP firmware used for passive packet sniffing |
| `firmware/source/ncp-uart-sw_7.4.3.0_115200.gbl` | Tested EmberZNet 7.4.3 coordinator/recovery firmware, 115200 baud, software flow control |

## 1. Install Node.js

`ember-zli` is a Node.js command-line application, so install Node.js first.

On Windows with `winget`:

```powershell
winget install OpenJS.NodeJS.LTS
```

Alternatively, download the current Node.js LTS installer from the official Node.js website.

After installation, close and reopen PowerShell and verify:

```powershell
node --version
npm --version
```

Both commands should print a version number.

## 2. Install ember-zli

Install `ember-zli` globally with npm:

```powershell
npm install -g ember-zli
```

Verify the installation:

```powershell
ember-zli --version
```

Show available commands if needed:

```powershell
ember-zli --help
```

The passive capture command used by this guide is:

```powershell
ember-zli sniff
```

If PowerShell reports that `ember-zli` is not recognized after installation, close and reopen PowerShell so the updated npm PATH is loaded.

Official ember-zli project: https://github.com/Nerivec/ember-zli

## 3. Install Tera Term

Download and install **Tera Term** for Windows.

Tera Term is used to access the Gecko bootloader over the CH340 serial port and transfer `.gbl` firmware using XMODEM.

Connect the dongle and open:

```text
Device Manager -> Ports (COM & LPT)
```

The tested adapter appeared as:

```text
USB-SERIAL CH340 (COM4)
```

Your COM number may be different.

## 4. Enter Gecko bootloader mode

The tested board has `BOOT` and `nRST` / reset controls.

Use this sequence:

1. Press and hold `BOOT`.
2. While holding `BOOT`, press and release `nRST`.
3. Release `BOOT`.

```text
Hold BOOT
  -> press nRST
  -> release nRST
  -> release BOOT
```

## 5. Open the bootloader in Tera Term

Start Tera Term and select **Serial**.

Select the CH340 COM port and configure:

```text
Baud rate: 115200
Data: 8 bit
Parity: none
Stop: 1 bit
```

Hardware RTS/CTS is not required for the bootloader transfer.

If the Gecko bootloader is active, its menu/prompt should appear in Tera Term. If the normal application starts instead, repeat the BOOT + nRST sequence.

## 6. Flash the passive sniffer firmware

The sniffer firmware is:

```text
firmware/source/SnifferNCP.gbl
```

With the Gecko bootloader open in Tera Term:

1. Select the bootloader menu option that starts a firmware upload / XMODEM receive operation.
2. In Tera Term open:

```text
File -> Transfer -> XMODEM -> Send
```

3. Select:

```text
firmware/source/SnifferNCP.gbl
```

4. Start the transfer.
5. Wait until the transfer is completely finished.
6. Do not unplug or reset the dongle during the upload.
7. If the firmware does not start automatically after a successful transfer, press `nRST` once.

## 7. Sniffer firmware configuration

The custom firmware used for this setup was built in **Simplicity Studio 6** from the Silicon Labs `Zigbee - NCP UART HW` example for `EFR32MG21A020F768IM32`, with `Zigbee -> Utility -> Manufacturing Library` enabled.

Working UART configuration:

```text
USART0
TX: PB01
RX: PB00
Baud: 115200
Flow control: software
RTS/CTS: disabled
```

The tested sniffer NCP reported:

```text
NCP: 9.1.1 [GA]
EZSP protocol: 19
```

## 8. Start a passive capture

Close Tera Term and any other application using the dongle COM port.

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
Output: PCAP
```

Then select the IEEE 802.15.4 / Zigbee channel you want to monitor.

For the tested Schneider PowerTag capture:

```text
Channel: 11
TX power: 5
```

A successful startup should report the NCP/EZSP version, start MFGLIB, and end with:

```text
Sniffing started.
```

Leave the command running while performing the Zigbee activity you want to capture. Stop it with:

```text
Ctrl+C
```

The resulting PCAP can then be opened in Wireshark.

## 9. Restore coordinator firmware

A tested recovery firmware is included in the repository:

```text
firmware/source/ncp-uart-sw_7.4.3.0_115200.gbl
```

This is an EmberZNet `7.4.3.0` NCP image configured for `115200` baud and software UART flow control.

### Step 1 - Stop the sniffer

If `ember-zli sniff` is running, stop it with:

```text
Ctrl+C
```

Close any program using the dongle COM port.

### Step 2 - Enter Gecko bootloader

Use exactly the same bootloader sequence:

```text
Hold BOOT
  -> press nRST
  -> release nRST
  -> release BOOT
```

### Step 3 - Connect with Tera Term

Open Tera Term, select the CH340 COM port, and use:

```text
115200 baud
8 data bits
no parity
1 stop bit
```

Confirm that the Gecko bootloader menu/prompt is visible.

### Step 4 - Start XMODEM upload

Select the Gecko bootloader menu option that starts firmware upload / XMODEM receive mode.

Then in Tera Term select:

```text
File -> Transfer -> XMODEM -> Send
```

Choose:

```text
firmware/source/ncp-uart-sw_7.4.3.0_115200.gbl
```

Wait for the transfer to finish completely.

### Step 5 - Restart the dongle

After a successful upload, allow the dongle to reboot. If necessary, press `nRST` once.

The dongle is now back on the tested EmberZNet 7.4.3 coordinator/NCP firmware rather than the custom MFGLIB sniffer firmware.

### Switching back to sniffer mode later

There is no need to rebuild the firmware. Enter the Gecko bootloader again and flash:

```text
firmware/source/SnifferNCP.gbl
```

Therefore the same dongle can be switched between:

```text
SnifferNCP.gbl
        <->
ncp-uart-sw_7.4.3.0_115200.gbl
```

using the same BOOT/nRST + Tera Term + XMODEM procedure.

## Troubleshooting

### Tera Term shows nothing

Check the COM port and `115200` baud. Make sure another program is not using the serial port.

### Normal firmware starts instead of the bootloader

Repeat the sequence carefully:

```text
Hold BOOT -> press/release nRST -> release BOOT
```

### XMODEM transfer does not start

Make sure the Gecko bootloader has first been put into its firmware-upload/XMODEM receive mode before selecting XMODEM Send in Tera Term.

### ember-zli cannot open the COM port

Close Tera Term and any other serial terminal before running:

```powershell
ember-zli sniff
```

### node, npm or ember-zli is not recognized

Close and reopen PowerShell after installation and check:

```powershell
node --version
npm --version
ember-zli --version
```

### Sniffer starts but receives no packets

Verify that the selected channel matches the Zigbee network or device being monitored.

### MFGLIB does not start

Make sure the custom firmware is `SnifferNCP.gbl` from this repository. MFGLIB support is required for this capture method.

## What this setup can be used for

With the dongle running the sniffer firmware, it can passively observe IEEE 802.15.4 / Zigbee traffic on a selected channel without joining the network or acting as the coordinator.

Typical uses include:

- capturing raw IEEE 802.15.4 and Zigbee frames to PCAP files;
- opening captures in Wireshark for packet-by-packet protocol analysis;
- observing device pairing, joining and commissioning exchanges;
- comparing traffic produced by different coordinators, gateways or devices;
- debugging cases where a Zigbee device is transmitting but an application or coordinator does not process the frames as expected;
- studying Zigbee Green Power and other IEEE 802.15.4-based traffic visible on the selected channel;
- investigating frame structure, addressing, counters, timing and retransmissions;
- performing interoperability and reverse-engineering research on hardware you are authorized to test.

The sniffer records what is transmitted over the air. It does not automatically decrypt protected Zigbee payloads; encrypted traffic can only be interpreted when the required security information is available separately.

Because the repository also includes the tested coordinator/recovery firmware, the same dongle can be restored after capturing and used again as an EFR32MG21 Zigbee coordinator.

## Disclaimer

This guide documents development and interoperability work performed on owned hardware. Firmware images are hardware-specific. Verify that the target dongle uses the expected `EFR32MG21A020F768IM32` hardware and UART configuration before flashing. Interrupting a firmware transfer or flashing an incompatible image can leave the dongle unusable until it is recovered through an appropriate bootloader or debug interface.