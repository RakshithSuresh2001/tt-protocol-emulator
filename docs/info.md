## How it works

A small array of programmable state machines drives the I/O pins. Each state machine runs a short program
(set and wait on pins, shift bits in and out, jump, delay) with its own clock divider and pin mapping, so
UART, SPI and I2C are implemented as programs rather than fixed hardware. New protocols can be added after
fabrication by loading new programs.

## How to test

Load a program and configuration over the SPI configuration port, then connect the mapped pins to a UART,
SPI or I2C device and compare the traffic with the expected protocol. Example programs and an assembler are
in the repository.

## External hardware

None required. Any UART, SPI or I2C device, or a logic analyzer, can be used to observe the protocols.
