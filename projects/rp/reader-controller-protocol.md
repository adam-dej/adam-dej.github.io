---
layout: default
title: Controller ↔ Reader Protocol
---

# Controller ↔ Reader Protocol

Controller and Reader are interconnected using a simple RS-232 based interface capable of supplying power to the Reader, resetting it and switching it to 'Firmware Upgrade' mode.

## Physical layer

Physically on the Reader and on the Controller there are RJ11 (6P6C) connectors. The Reader and Controller are then interconnected with 6-wire standard telephone cable. Maximum length of the cable is 3m.

#### Pinout

Pin | Function
----|---------
 1  | RST
 2  | Vcc
 3  | RxD
 4  | TxD
 5  | GND
 6  | Boot

Note that RxD and TxD pins are relative to the Reader, therefore TxD pin is the pin Reader transmits on. It should be connected to the RxD pin of the Controller.

#### Signalling

GND and Vcc are the power supply pins. The voltage may be anywhere between +4 to +25 volts, and the Controller must have the capability to supply at least 100mA of current at +5V (less at higher voltages).

RST and Boot pins are driven by the Controller. Logical signalling level on those pins is +3.3V. RST is active LOW, idle HIGH. Boot is idle LOW, active HIGH.
LOW pulse with the duration of at least (TODO figure out the exact time)ms will generate reset of the Reader.
If Boot signal is active during power-up or reset the Reader enters the 'Bootloader mode'. Read documentation for the [Reader](/projects/rp/reader-doc.html) for more info on the 'Bootloader mode'.

RxD and TxD pins are RS-232 compatible UART communication lines, with voltage levels as specified by RS-232 standard.
Used baud rate is `38 400 baud`, one end bit, even parity.

## Protocol layer

### General idea

This is a bidirectional asynchronous protocol that must guarantee delivery of each message, although it does not have to guarantee the order of delivery. Either side may initiate communication and the receiving side must always acknowledge every received packet. If the acknowledgment is not received within a certain time the message will be resent by the initiating side. If the acknowledgment is not received after `r_max` retries the initiating side will switch to 'communication outage' mode, behavior of which is different for the Reader and for the Controller.

### Packet format

Each packet has the following format:

Size | Name
-----|---------------------------
  1  | `packet_id`
  1  | `transx_num`
  1  | `seq_id`
  1  | `payload_size`
  n  | `payload`
  1  | `checksum`

  - **`packet_id`** identifies the type of the packet.
  - **`transx_num`** (Transmission attempt number) is the number of the attempt to transmit this packet.
  - **`seq_id`** (Sequential identifier) is an unique identifier of this packet, to distinguish which packet is being ACK-ed. It may be reused once the packet delivery has succeeded.
  - **`payload_size`** Size of the `Payload` part of the packet, not packet header and checksum.
  - **`payload`** Data of given or variable length specific for this type of packet.
  - **`checksum`** XOR of all bytes in this packet.

### Communication rules

  - Device must always acknowledge the reception of a packet.
  - Device must acknowledge, but take no action, upon reception of packet with the same `seq_id` and higher `transx_num` than already processed packet.
  - Device must be able to re-transmit every packet (except the acknowledge packet) that it has previously transmitted and has not yet received acknowledgment of its reception.
  - Device must select appropriate `seq_id` for each packet. The `seq_id` must be different from `seq_id` of each transmitted packet by the device that has not yet been acknowledged.
  - Device must automatically re-transmit every packet that has not been acknowledged after `t_rtx` from its transmission.
  - Device must enter 'communication outage' mode if it does not receive acknowledgment of a packet within `t_rtx` from its last transmission after `n_rtx` re-transmission attempts.

Defined values are:

  - `t_rtx` = 100ms
  - `n_rtx` = 5

### Packet types

#### `packet_ack`

**Header:**

Field         | Value
--------------|-------
Packet ID     | 0
Payload size  | 0

**Payload:**

None.

**Direction:**

Both Controller and Reader may send this packet.

**Description:**

This is an 'acknowledged' response which must be sent as a reply to every received transmission.

This packet serves a special purpose, and values in fields `transx_num` and `seq_id` follow different rules: They are the same as values of `transx_num` and `seq_id` of the packet being acked. This packet carries no payload.


#### `packet_init`

**Header:**

Field         | Value
--------------|-------
Packet ID     | 1
Payload size  | 0

**Payload:**

None.

**Direction:**

Only Controller may send this packet.

**Description:**

This packet initializes the Reader. If the Reader is already initialized it has no effect, but Reader must still ack it.

This is the first packet that must be sent after Reader powerup. Reader will ignore any packet before this one. After reception of this it will turn on transmitting part of its circuitry and ack this packet. It will also start polling for cards and commence normal operation. Reception of this packet will also change state of the UI from `ui_controller_out` to `ui_idle`. After the Reader is initialized this packet may also be used as a form of communication check.

#### `packet_card`

**Header:**

Field         | Value
--------------|-------
Packet ID     | 2
Payload size  | 4, 7 or 10

**Payload:**

Received ID of the RFID card.

**Direction:**

Only Reader may send this packet.

**Description:**

This packet tells the Controller that the Reader has successfully read an ID of a card.

#### `packet_ui`

**Header:**

Field         | Value
--------------|-------
Packet ID     | 3
Payload size  | 1

**Payload:**

One byte with id of UI state to switch to.

**Direction:**

Only Controller may send this packet.

**Description:**

This packet switches UI of the reader to some other state.

For list of defined states and their numbers please see documentation for the [Reader](/projects/rp/reader-doc.html).

#### `packet_comm_error`

**Header:**

Field         | Value
--------------|-------
Packet ID     | 4
Payload size  | 0

**Payload:**

None.

**Direction:**

Only Reader may send this packet.

**Description:**

This packet is sent when the Reader enters a 'communication outage' mode. There is a chance that only communication in direction Controller → Reader is broken, and this packet therefore may get to the Controller informing it about the problem.
