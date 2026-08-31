# Firmware

Place the tested custom sniffer firmware in this directory as:

```text
PowerTagSnifferNCP.gbl
```

Expected final path:

```text
firmware/PowerTagSnifferNCP.gbl
```

The firmware is a Silicon Labs Zigbee NCP UART build for the tested SZKOSTON-style EFR32MG21 dongle with MFGLIB / Manufacturing Library enabled.

Tested hardware configuration:

```text
MCU: EFR32MG21A020F768IM32
UART: USART0
TX: PB01
RX: PB00
Baud: 115200
Flow control: software / XON-XOFF
RTS/CTS: disabled
```

The firmware binary is not included yet. Add the verified `.gbl` file here before following the flashing instructions in the main README.

Keep a copy of the original/recovery NCP firmware separately before flashing.