# Lab 8: CAN-bus Telemetry and Remote Motor Control (TM4C123)

Two TM4C123 LaunchPads (on EduARM4 boards) communicate over a CAN bus at 500 kbit/s using the built-in CAN0 controller and an MCP2551 transceiver module on each board. Both programs use the default startup file and no interrupts (polling, 10 ms loop).

| File | Board | Role |
|---|---|---|
| `sensor_node/main.c` | Node 1 | Reads inputs, sends the CAN frame, shows status |
| `actuator_node/main.c` | Node 2 | Receives the frame, drives two motors, shows status |

## Sensor node

- Potentiometer on PE3 (ADC0, AIN0), converted to 0-100 %.
- DHT11 on PD2, read every 2 s, whole degrees C. On a failed read the last good value is kept and a fault flag is sent.
- SW1 (PF4) and SW2 (PF0, unlocked in code): each press toggles direction 1 / direction 2 between CW and ACW.
- Sends one frame every 100 ms.
- LCD line 1: temperature and pot %. Line 2: SW1 and SW2 directions. A failed DHT11 read shows an error code in place of the temperature: E1 no response, E2 line stuck low, E3 line stuck high, E4 bit timing, E5 checksum.
- Recovers automatically from CAN bus-off.

## Actuator node

- Motor-1 speed = pot value.
- Motor-2 speed = 0 % at 20 C or below, 100 % at 50 C or above, linear from 30 % to 100 % in between. Forced to 0 % when the sensor-fault flag is set.
- Motor-1 and Motor-2 directions follow direction 1 and direction 2 from the frame.
- PWM on PWM1 generator 3 at 1 kHz; speed 0 switches the PWM output off.
- No frame for more than 500 ms: both motors stop and the LCD shows `NO CAN DATA` / `Motors stopped`.
- LCD line 1: Motor-1 speed and direction. Line 2: Motor-2 speed and direction.
- Recovers automatically from CAN bus-off.

## CAN frame

ID `0x100` (11-bit), DLC 5, every 100 ms, message object 1 on both nodes.

| Byte | Content |
|---|---|
| 0 | Potentiometer, 0-100 % |
| 1 | Temperature (C), high byte |
| 2 | Temperature (C), low byte |
| 3 | Flags: bit 0 = direction 1, bit 1 = direction 2 (0 = CW, 1 = ACW), bit 2 = sensor fault |
| 4 | Rolling counter |

Bit timing at 16 MHz: `CAN0_BIT_R = 0x2B41` (prescaler 2, TSEG1 = 12, TSEG2 = 3, SJW = 2, 16 quanta per bit, sample point about 81 %). Must be identical on both nodes.

## Wiring

**Both nodes**

| Tiva | Connection |
|---|---|
| PB0-PB7 | LCD D0-D7 |
| PA6 / PA7 | LCD RS / E |
| PE5 | MCP2551 TX |
| PE4 | MCP2551 RX through 1 k ohm, with 2.2 k ohm from PE4 to GND |
| VBUS / GND | MCP2551 VCC / GND |

- LCD RW is tied to GND on the board. EduARM4 jumper J5 shorted on pins 3-4 selects the LCD.
- Bus: CANH to CANH, CANL to CANL, 120 ohm termination at each end (about 60 ohm CANH to CANL with power off).
- All grounds common: both Tivas, both MCP2551 modules, the L298N and the motor supply.

**Sensor node**

| Tiva | Connection |
|---|---|
| PE3 | Potentiometer wiper (ends to 3.3 V and GND) |
| PD2 (J3 pin 5) | DHT11 DATA (internal pull-up enabled); DHT11 VCC 5 V |
| PF4 / PF0 | Onboard SW1 / SW2 |

**Actuator node (L298N)**

| Tiva | L298N | Function |
|---|---|---|
| PF2 | ENA (jumper removed) | Motor-1 speed |
| PA2 / PA3 | IN1 / IN2 | Motor-1 CW / ACW |
| PF3 | ENB (jumper removed) | Motor-2 speed |
| PA4 / PA5 | IN3 / IN4 | Motor-2 CW / ACW |

OUT1-OUT2 drive Motor-1 and OUT3-OUT4 drive Motor-2. +12V comes from an external supply; the L298N 5V pin is not connected.

