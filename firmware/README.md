# Firmware

The tested custom sniffer firmware for this repository is:

```text
SnifferNCP.gbl
```

Expected path:

```text
firmware/SnifferNCP.gbl
```

Firmware SHA-256:

```text
5c1357bce9c69b8ca66b63d7669b86c4365457d1a1dfd8db36a668abaf7f06e0
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

Keep a copy of the original/recovery NCP firmware separately before flashing.