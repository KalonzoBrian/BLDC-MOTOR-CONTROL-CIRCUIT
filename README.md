# Sensorless Brushless DC (BLDC) Motor Control with Arduino

A complete DIY Electronic Speed Controller (ESC) implementation for sensorless BLDC motor control using Arduino UNO, featuring back-EMF sensing and analog comparator-based commutation.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Components Required](#components-required)
- [Hardware Design](#hardware-design)
  - [Circuit Architecture](#circuit-architecture)
  - [Power Stage](#power-stage)
  - [Sensing Stage](#sensing-stage)
  - [Control Stage](#control-stage)
- [Schematic Connections](#schematic-connections)
- [Software Architecture](#software-architecture)
  - [Timer Configuration](#timer-configuration)
  - [Analog Comparator Setup](#analog-comparator-setup)
  - [Commutation Sequences](#commutation-sequences)
- [Code Functions Reference](#code-functions-reference)
- [Motor Operation](#motor-operation)
- [Speed Control](#speed-control)
- [Getting Started](#getting-started)
- [Technical Details](#technical-details)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Project Overview

This project implements a **sensorless BLDC motor controller** using Arduino UNO as the brain of the system. Unlike traditional ESCs that require external motor sensors (Hall sensors), this design uses **Back-EMF (BEMF) sensing** to determine rotor position and timing for commutation.

### Key Features:

- **Sensorless Control**: Back-EMF based rotor position detection
- **Arduino UNO Based**: ATmega328P microcontroller
- **Analog Comparator**: Efficient zero-crossing detection
- **PWM Control**: ~31 kHz PWM frequency on high-side MOSFETs
- **Speed Control**: Push-button based speed adjustment
- **Minimal Hardware**: Optimized component count
- **Commutation Timing**: Six-step commutation with BEMF monitoring

---

## Components Required

### Microcontroller & Control
| Component | Quantity | Notes |
|-----------|----------|-------|
| Arduino UNO Board | 1 | Based on ATmega328P |
| Breadboard | 1 | For prototyping |
| Jumper Wires | Multiple | Various lengths |

### Motor & Drive
| Component | Quantity | Specifications |
|-----------|----------|---|
| Brushless DC (BLDC) Motor | 1 | 12V rated |
| 06N03LA N-type MOSFET | 6 | IR2101 gate driver control |

### Gate Drivers
| Component | Quantity | Specifications |
|-----------|----------|---|
| IR2101 Gate Driver IC | 3 | One per motor phase (A, B, C) |

### Passive Components
| Component | Quantity | Specifications |
|-----------|----------|---|
| 33kΩ Resistor | 6 | Voltage dividers (3 motor phases) + Virtual neutral (3) |
| 10kΩ Resistor | 3 | Voltage dividers for BEMF sensing |
| 10Ω Resistor | 6 | Gate driver current limiting |
| IN4148 Diode | 3 | Bootstrap capacitor protection |
| 10µF Capacitor | 3 | IR2101 bootstrap capacitors |
| 2.2µF Capacitor | 3 | Gate driver bypass capacitors |

### Power & Control Input
| Component | Quantity | Specifications |
|-----------|----------|---|
| 12V Power Source | 1 | 10-20A capable (motor dependent) |
| Push Button | 2 | Momentary contact (Speed up / Speed down) |

---

## Hardware Design

### Circuit Architecture

The control circuit consists of three main stages:

1. **Power Stage**: Three-phase inverter with 6 MOSFETs (2 per phase)
2. **Sensing Stage**: BEMF detection via voltage dividers
3. **Control Stage**: Arduino with analog comparator and PWM generation

### Power Stage

**Three-Phase Inverter Configuration:**

Each motor phase (A, B, C) uses a half-bridge topology:
- **High-side MOSFET** (HS): Pulled high by IR2101 high-side driver
- **Low-side MOSFET** (LS): Pulled low by IR2101 low-side driver
- **Bootstrap Capacitor** (10µF): Supplies gate drive voltage for high-side MOSFET

```
       +12V
        |
      [HSMOSFET A]
        |----------- Phase A Output
      [LSMOSFET A]
        |
       GND
```

**IR2101 Gate Driver Details:**

The IR2101 provides galvanic isolation between phase node and gate signals:
- **HIN**: PWM input for high-side gate (from Arduino)
- **LIN**: Direct high-side/low-side control (tied appropriately per sequence)
- **Bootstrap Charge**: Bootstraps from phase node during low-side conduction
- **Dead Time**: Inherent to prevent shoot-through

**Bootstrap Circuit:**
- Capacitor charges through IN4148 diode when low-side MOSFET is ON
- Diode prevents discharge back through the low-side driver
- Typical bootstrap supply voltage: 12V + phase voltage rise

### Sensing Stage

**Back-EMF Voltage Dividers:**

Three voltage divider networks sample BEMF from motor phases:

```
Phase A (12V) ---[33kΩ]--- AIN0 (Arduino Pin 6) ---[33kΩ]--- GND
              (sampled at pin 7 AIN1)

Phase B (12V) ---[33kΩ]--- A2 (Arduino Pin A2) ---[33kΩ]--- GND
              (via ADC2 as comparator input)

Phase C (12V) ---[33kΩ]--- A3 (Arduino Pin A3) ---[33kΩ]--- GND
              (via ADC3 as comparator input)
```

**Virtual Neutral Point:**

Three 33kΩ resistors create a virtual neutral reference:
- Connected to Arduino Pin 6 (AIN0) - Analog Comparator positive input
- Provides reference voltage at motor neutral point (~6V for 12V system)

**Function:**
- BEMF signals oscillate around the neutral point
- Neutral point provides a stable reference
- Zero-crossing detection occurs when BEMF crosses neutral

### Control Stage

**Arduino Pin Assignments:**

| Pin | Function | Type | Notes |
|-----|----------|------|-------|
| 3 | Phase A Low-side Control | Digital Output | PORTD bit 3 |
| 4 | Phase B Low-side Control | Digital Output | PORTD bit 2 |
| 5 | Phase C Low-side Control | Digital Output | PORTD bit 0 |
| 6 (AIN0) | Neutral Point Reference | Analog Input | Comparator + |
| 7 (AIN1) | Phase A BEMF | Analog Input | Comparator - (initial) |
| 9 (OC1A) | Phase B High-side PWM | PWM Output | Timer1 |
| 10 (OC1B) | Phase C High-side PWM | PWM Output | Timer1 |
| 11 (OC2A) | Phase A High-side PWM | PWM Output | Timer2 |
| A0 | Speed Up Button | Digital Input | Internal pull-up |
| A1 | Speed Down Button | Digital Input | Internal pull-up |
| A2 (ADC2) | Phase B BEMF | Analog Input | Comparator - (dynamic) |
| A3 (ADC3) | Phase C BEMF | Analog Input | Comparator - (dynamic) |

---

## Schematic Connections

### Detailed Connection Map

#### Arduino to IR2101 Gate Drivers:

```
Arduino Pin 9 (OC1A)  --> IR2101_A HIN
Arduino Pin 10 (OC1B) --> IR2101_B HIN
Arduino Pin 11 (OC2A) --> IR2101_C HIN

Arduino Pin 3 --> IR2101_A LIN
Arduino Pin 4 --> IR2101_B LIN
Arduino Pin 5 --> IR2101_C LIN

Arduino GND --> All IR2101 VSS pins
12V Source --> All IR2101 VCC pins (via IR2101)
```

#### Motor Phase Connections:

```
Motor Phase A --> IR2101_A Output (OUT pin)
Motor Phase B --> IR2101_B Output (OUT pin)
Motor Phase C --> IR2101_C Output (OUT pin)

Motor Neutral --> Virtual Neutral Point (Arduino Pin 6)
```

#### BEMF Sensing:

```
Phase A Node ---[33kΩ]---+---[33kΩ]--- GND
                          |
                    Arduino Pin 7 (AIN1)

Phase B Node ---[33kΩ]---+---[33kΩ]--- GND
                          |
                    Arduino Pin A2

Phase C Node ---[33kΩ]---+---[33kΩ]--- GND
                          |
                    Arduino Pin A3

Virtual Neutral: All three 33kΩ resistors (non-motor side) --> Arduino Pin 6 (AIN0)
```

#### Control Inputs:

```
Speed Up Button:   One pin to Arduino A0, other to GND
Speed Down Button: One pin to Arduino A1, other to GND
(Buttons configured with internal pull-ups)
```

---

## Software Architecture

### Timer Configuration

**Timer1 (OC1A on Pin 9, OC1B on Pin 10):**

```cpp
TCCR1A = Configuration register A
TCCR1B = 0x01  // Clock source: clkI/O / 1 (no prescaling)
```

- **PWM Frequency**: ~31 kHz (16 MHz / 256 / 2)
- **Resolution**: 8-bit (256 levels, 0-255)
- **Registers Modified**: OCR1A (Pin 9), OCR1B (Pin 10)

**Timer2 (OC2A on Pin 11):**

```cpp
TCCR2A = Configuration register A
TCCR2B = 0x01  // Clock source: clkI/O / 1 (no prescaling)
```

- **PWM Frequency**: ~31 kHz (16 MHz / 256 / 2)
- **Resolution**: 8-bit (256 levels, 0-255)
- **Register Modified**: OCR2A (Pin 11)

**Duty Cycle Control:**

The duty cycle directly controls motor speed. Values range from:
- **PWM_MIN_DUTY**: 50 (minimum safe commutation speed)
- **PWM_MAX_DUTY**: 255 (maximum speed)
- **PWM_START_DUTY**: 100 (initial ramp speed)

```cpp
#define PWM_MAX_DUTY   255
#define PWM_MIN_DUTY   50
#define PWM_START_DUTY 100
```

### Analog Comparator Setup

**Analog Comparator Function:**

The ATmega328P analog comparator compares:
- **Positive Input (AIN0)**: Arduino Pin 6 - Virtual neutral point
- **Negative Input**: Switchable between AIN1 (Pin 7), ADC0, ADC1, ADC2, ADC3, ADC4, ADC5

**BEMF Zero-Crossing Detection:**

When BEMF crosses the neutral point:
1. Comparator output (ACO) toggles
2. Interrupt is triggered (rising or falling edge)
3. ISR handles commutation sequence advancement
4. Next BEMF phase is selected for monitoring

**Comparator Configuration:**

```cpp
ACSR = 0x10  // Initial: disable interrupt, clear flag
ACSR |= 0x08 // Enable comparator interrupt (after motor start)
ACSR |= 0x03 // Interrupt on rising edge (both edges)
ACSR &= ~0x01 // Interrupt on falling edge only
```

**ADCSRB Register (Comparator MUX Control):**

```cpp
ADCSRB = (0 << ACME)  // Select AIN1 (Pin 7) as negative input
ADCSRB = (1 << ACME)  // Select ADC channel (set by ADMUX)
```

**ADMUX Register (ADC Channel Selection):**

```cpp
ADMUX = 2  // Select ADC2 (Pin A2) for Phase B BEMF
ADMUX = 3  // Select ADC3 (Pin A3) for Phase C BEMF
```

### Commutation Sequences

**Six-Step Commutation Table:**

The BLDC motor requires precise coil energization sequencing. Each step represents a 60° electrical rotation:

| Step | Phase A | Phase B | Phase C | BEMF Monitor | Next Event |
|------|---------|---------|---------|--------------|-----------|
| 0 | AH+BL- | OFF | OFF | C Rising | → Step 1 |
| 1 | AH+CL- | OFF | OFF | B Falling | → Step 2 |
| 2 | BH+CL- | OFF | OFF | A Rising | → Step 3 |
| 3 | BH+AL- | OFF | OFF | C Falling | → Step 4 |
| 4 | CH+AL- | OFF | OFF | B Rising | → Step 5 |
| 5 | CH+BL- | OFF | OFF | A Falling | → Step 0 |

**Legend:**
- **H+**: High-side MOSFET ON (with PWM)
- **L-**: Low-side MOSFET ON (pulled to GND)
- **Rising/Falling**: BEMF zero-crossing direction to detect

**Hardware Implementation:**

Each commutation step is implemented via:
1. **PORTD Control**: Sets low-side MOSFET outputs (pins 3, 4, 5)
2. **TCCR Registers**: Routes PWM to correct high-side MOSFET
3. **Comparator Setup**: Selects next BEMF phase to monitor

```cpp
void AH_BL() {           // Step 0: A high + B low
  PORTD &= ~0x28;        // Clear bits 5 and 3
  PORTD |=  0x10;        // Set bit 4 (Phase B low-side active)
  TCCR1A = 0;            // Disable Timer1 PWM
  TCCR2A = 0x81;         // Enable Timer2 PWM on pin 11 (Phase A high)
}
```

---

## Code Functions Reference

### Core Functions

#### `setup()`
Initializes all hardware components on startup:
- Configures Arduino pins as inputs/outputs
- Sets timer modes and clock sources
- Initializes analog comparator
- Enables pull-up resistors on push buttons

**Key Operations:**
```cpp
DDRD  |= 0x38  // Pins 3, 4, 5 as outputs
DDRB  |= 0x0E  // Pins 9, 10, 11 as outputs
TCCR1A = 0; TCCR1B = 0x01  // Timer1 setup
TCCR2A = 0; TCCR2B = 0x01  // Timer2 setup
ACSR = 0x10    // Comparator setup
```

#### `loop()`
Main program loop handling motor startup and speed control:

1. **Motor Start Sequence** (~500ms):
   - Applies forced commutation pattern
   - Gradually decreases delay between steps (ramps motor speed)
   - Starts at 5ms delay, decrements by 20µs per step
   - Transitions to BEMF sensing at ~100µs delay

2. **Speed Control**:
   - Monitors SPEED_UP button → increment duty cycle
   - Monitors SPEED_DOWN button → decrement duty cycle
   - Updates PWM registers every 100ms while button pressed
   - Constrains speed between PWM_MIN_DUTY and PWM_MAX_DUTY

```cpp
while(1) {
  // Accelerate
  while(!(digitalRead(SPEED_UP)) && motor_speed < PWM_MAX_DUTY) {
    motor_speed++;
    SET_PWM_DUTY(motor_speed);
    delay(100);
  }
  // Decelerate
  while(!(digitalRead(SPEED_DOWN)) && motor_speed > PWM_MIN_DUTY) {
    motor_speed--;
    SET_PWM_DUTY(motor_speed);
    delay(100);
  }
}
```

#### `ISR(ANALOG_COMP_vect)`
Analog Comparator Interrupt Service Routine - triggered on BEMF zero-crossing:

**BEMF Debouncing:**
- Reads BEMF signal 10 times to confirm valid transition
- Resets debounce counter if signal unstable
- Prevents false commutations from noise

**Commutation:**
- Calls `bldc_move()` to execute next commutation step
- Increments `bldc_step` (0-5 cycle)
- Triggers next BEMF phase monitoring

```cpp
ISR(ANALOG_COMP_vect) {
  for(i = 0; i < 10; i++) {
    if(bldc_step & 1) {
      if(!(ACSR & 0x20)) i -= 1;  // Wait for expected transition
    } else {
      if((ACSR & 0x20)) i -= 1;
    }
  }
  bldc_move();
  bldc_step++;
  bldc_step %= 6;
}
```

#### `bldc_move()`
Executes the appropriate commutation sequence based on current step:

Routes PWM to correct high-side MOSFET and selects next BEMF phase to monitor:

```cpp
void bldc_move() {
  switch(bldc_step) {
    case 0: AH_BL(); BEMF_C_RISING();  break;   // A+ B- → Watch C cross
    case 1: AH_CL(); BEMF_B_FALLING(); break;   // A+ C- → Watch B cross
    case 2: BH_CL(); BEMF_A_RISING();  break;   // B+ C- → Watch A cross
    case 3: BH_AL(); BEMF_C_FALLING(); break;   // B+ A- → Watch C cross
    case 4: CH_AL(); BEMF_B_RISING();  break;   // C+ A- → Watch B cross
    case 5: CH_BL(); BEMF_A_FALLING(); break;   // C+ B- → Watch A cross
  }
}
```

### Commutation Helper Functions

These six functions implement the three-phase switching patterns:

#### `AH_BL()` - Step 0: Phase A High + Phase B Low
```cpp
PORTD &= ~0x28;  // Clear Phase B & C low-side outputs
PORTD |=  0x10;  // Set Phase B low-side output
TCCR1A = 0;      // Disable Timer1 PWM
TCCR2A = 0x81;   // Enable Timer2 PWM (Phase A high on pin 11)
```

#### `AH_CL()` - Step 1: Phase A High + Phase C Low
```cpp
PORTD &= ~0x30;  // Clear Phase B & C low-side outputs
PORTD |=  0x08;  // Set Phase C low-side output
TCCR1A = 0;      // Disable Timer1 PWM
TCCR2A = 0x81;   // Enable Timer2 PWM (Phase A high on pin 11)
```

#### `BH_CL()` - Step 2: Phase B High + Phase C Low
```cpp
PORTD &= ~0x30;  // Clear Phase B & C low-side outputs
PORTD |=  0x08;  // Set Phase C low-side output
TCCR2A = 0;      // Disable Timer2 PWM
TCCR1A = 0x21;   // Enable Timer1 PWM (Phase B high on pin 10)
```

#### `BH_AL()` - Step 3: Phase B High + Phase A Low
```cpp
PORTD &= ~0x18;  // Clear Phase A & B low-side outputs
PORTD |=  0x20;  // Set Phase A low-side output
TCCR2A = 0;      // Disable Timer2 PWM
TCCR1A = 0x21;   // Enable Timer1 PWM (Phase B high on pin 10)
```

#### `CH_AL()` - Step 4: Phase C High + Phase A Low
```cpp
PORTD &= ~0x18;  // Clear Phase A & B low-side outputs
PORTD |=  0x20;  // Set Phase A low-side output
TCCR2A = 0;      // Disable Timer2 PWM
TCCR1A = 0x81;   // Enable Timer1 PWM (Phase C high on pin 9)
```

#### `CH_BL()` - Step 5: Phase C High + Phase B Low
```cpp
PORTD &= ~0x28;  // Clear Phase B & C low-side outputs
PORTD |=  0x10;  // Set Phase B low-side output
TCCR2A = 0;      // Disable Timer2 PWM
TCCR1A = 0x81;   // Enable Timer1 PWM (Phase C high on pin 9)
```

### BEMF Sensing Functions

These functions dynamically select which motor phase's BEMF to monitor and set the comparator interrupt edge:

#### `BEMF_A_RISING()`
```cpp
void BEMF_A_RISING() {
  ADCSRB = (0 << ACME);    // Use AIN1 (Pin 7) as comparator input
  ACSR |= 0x03;            // Interrupt on rising edge (both bits set)
}
```

#### `BEMF_A_FALLING()`
```cpp
void BEMF_A_FALLING() {
  ADCSRB = (0 << ACME);    // Use AIN1 (Pin 7) as comparator input
  ACSR &= ~0x01;           // Interrupt on falling edge only
}
```

#### `BEMF_B_RISING()` / `BEMF_B_FALLING()`
```cpp
void BEMF_B_RISING() {
  ADCSRA = (0 << ADEN);    // Disable ADC module
  ADCSRB = (1 << ACME);    // Use ADC channel as comparator input
  ADMUX = 2;               // Select ADC2 (Pin A2)
  ACSR |= 0x03;            // Interrupt on rising edge
}
```

#### `BEMF_C_RISING()` / `BEMF_C_FALLING()`
```cpp
void BEMF_C_RISING() {
  ADCSRA = (0 << ADEN);    // Disable ADC module
  ADCSRB = (1 << ACME);    // Use ADC channel as comparator input
  ADMUX = 3;               // Select ADC3 (Pin A3)
  ACSR |= 0x03;            // Interrupt on rising edge
}
```

#### `SET_PWM_DUTY(byte duty)`
Updates PWM duty cycle for all three phases simultaneously:

```cpp
void SET_PWM_DUTY(byte duty) {
  if(duty < PWM_MIN_DUTY)  duty = PWM_MIN_DUTY;
  if(duty > PWM_MAX_DUTY)  duty = PWM_MAX_DUTY;
  
  OCR1A = duty;  // Timer1 channel A (Pin 9)
  OCR1B = duty;  // Timer1 channel B (Pin 10)
  OCR2A = duty;  // Timer2 channel A (Pin 11)
}
```

---

## Motor Operation

### Startup Sequence

The motor cannot start using BEMF detection alone (BEMF is zero at standstill). The code implements a forced-commutation startup:

**Phase 1: Forced Commutation Ramp (500ms)**
1. Motor coils energized in fixed sequence (not waiting for BEMF)
2. Delay between steps starts at 5,000µs (5ms)
3. Decreases by 20µs per step
4. Creates ramping motion that accelerates the rotor
5. Continues until delay reaches ~100µs

**Phase 2: BEMF Detection Takeover**
1. At sufficient speed, BEMF signals become detectable
2. Analog comparator interrupt enabled
3. Control transitions from forced commutation to zero-crossing detection
4. Motor speed continues to increase due to PWM duty cycle

**Phase 3: Closed-Loop Operation**
1. BEMF zero-crossing triggers commutation (approximately every 30-40µs at speed)
2. Motor speed responds to PWM duty cycle
3. Speed control via push buttons adjusts PWM duty cycle

### Back-EMF and Commutation Timing

**Back-EMF Generation:**

When a motor coil cuts through magnetic field lines, it generates an induced voltage (BEMF):

```
Commutation Pattern: A+ B- → A+ C- → B+ C- → B+ A- → C+ A- → C+ B-
                    (60°)  (60°)  (60°)  (60°)  (60°)  (60°)

Inactive Phase:       C      B      A      C      B      A
(BEMF Monitored)
```

**Zero-Crossing Detection:**

- BEMF oscillates around the neutral point as the rotor rotates
- It crosses zero when the rotor reaches 30° past the previous commutation
- Detecting this crossing indicates optimal time for next commutation
- Commutation occurs when BEMF crosses neutral (rising or falling)

**Commutation Advance:**

Proper commutation timing is critical:
- **Too early**: Motor loses torque and may stall
- **Too late**: Motor becomes inefficient and vibrates
- **Optimal**: 30° electrical after the previous commutation

The zero-crossing detection provides this 30° advance automatically.

---

## Speed Control

### Speed Control Mechanism

Motor speed is directly proportional to PWM duty cycle applied to high-side MOSFETs:

**Low Duty Cycle** (50-100):
- Brief high-side conduction window
- Lower average phase current
- Slower rotor acceleration
- Lower steady-state speed

**High Duty Cycle** (150-255):
- Extended high-side conduction window
- Higher average phase current
- Faster rotor acceleration
- Higher steady-state speed

### Push Button Interface

**Speed Up Button (Arduino Pin A0):**
- Connected to GND when pressed
- Pulled high via internal pull-up when released
- Increments `motor_speed` variable
- Maximum speed limited to PWM_MAX_DUTY (255)
- Update rate: 100ms per increment

**Speed Down Button (Arduino Pin A1):**
- Connected to GND when pressed
- Pulled high via internal pull-up when released
- Decrements `motor_speed` variable
- Minimum speed limited to PWM_MIN_DUTY (50)
- Update rate: 100ms per decrement

### PWM Duty Cycle Constraints

```cpp
#define PWM_MIN_DUTY 50    // Motor may stall below this
#define PWM_MAX_DUTY 255   // Maximum (100% duty)
#define PWM_START_DUTY 100 // Initial ramp speed

SET_PWM_DUTY(motor_speed);  // Applied simultaneously to all PWM channels
```

---

## Getting Started

### Prerequisites
- Arduino IDE installed on your computer
- Arduino UNO board
- USB cable for programming
- All components listed in [Components Required](#components-required)

### Assembly Instructions

1. **Breadboard Layout**:
   - Place IR2101 ICs on breadboard
   - Mount MOSFETs with appropriate spacing
   - Add decoupling capacitors near power inputs
   - Keep gate drive connections short to minimize noise

2. **Wiring Priority** (Solder in this order):
   - Power connections (12V and GND) - verify before powering on
   - Motor phase connections
   - Arduino pin connections
   - Bootstrap capacitors and diodes
   - Sensing network (resistor dividers)
   - Button connections

3. **Verification Checklist**:
   - [ ] No shorts between 12V and GND
   - [ ] All gate driver power connections solid
   - [ ] Phase connections to motor correct
   - [ ] BEMF sensing network properly wired
   - [ ] Arduino pins 9, 10, 11 connected to HIN inputs
   - [ ] Arduino pins 3, 4, 5 connected to LIN inputs
   - [ ] Push buttons properly connected to A0, A1 with GND

### Programming

1. Connect Arduino UNO to computer via USB
2. Open Arduino IDE
3. Select Board: "Arduino UNO"
4. Select COM Port for your Arduino
5. Copy the code from the project
6. Paste into Arduino IDE sketch window
7. Click Upload button
8. Wait for "Done uploading" message

### Initial Test Procedure

**Safety First**: Ensure motor is properly mounted and cannot eject.

1. Power on 12V supply (motor should not move yet)
2. Check that Arduino LED blinks (indicates sketch loaded)
3. Press Speed Up button once
4. Motor should begin startup sequence (~1 second ramp)
5. Motor should reach steady speed
6. Press Speed Up to increase speed (motor accelerates)
7. Press Speed Down to decrease speed (motor decelerates)
8. Motor should maintain smooth operation across speed range

### Troubleshooting Initial Issues

**Motor doesn't start:**
- Check all phase connections to motor
- Verify 12V supply voltage
- Measure voltage at Arduino pin 6 (should be ~6V)
- Check IR2101 outputs with oscilloscope

**Motor starts then stops:**
- BEMF sensing issue; check resistor dividers
- Verify Arduino pins 7, A2, A3 have clean signals
- Motor may be too large for power supply current capacity

**Motor vibrates or doesn't accelerate smoothly:**
- Check PWM_MIN_DUTY setting (increase if needed)
- Verify BEMF debounce works (reduce from 10 if too restrictive)
- Check for loose breadboard connections

---

## Technical Details

### ATmega328P Analog Comparator

**Register Reference:**

| Register | Bits | Function |
|----------|------|----------|
| ACSR | 7:6 | Analog Comparator Control |
| ACSR | 5 | Analog Comparator Output (ACO) |
| ACSR | 4 | Analog Comparator Interrupt Enable |
| ACSR | 3:2 | Interrupt Mode Select |
| ACSR | 0 | Analog Comparator Interrupt Flag |

**Interrupt Mode Encoding:**
- Bit 1:0 = 00: Comparator interrupt on low level output
- Bit 1:0 = 01: Comparator interrupt on any logical change
- Bit 1:0 = 10: Comparator interrupt on falling output edge
- Bit 1:0 = 11: Comparator interrupt on rising output edge

### Timer/PWM Modes

**Fast PWM Mode (used in this project):**
- Counter runs from 0x00 to 0xFF (8-bit)
- OCRx compared with counter value
- Output goes high when counter < OCRx, low when counter ≥ OCRx
- Results in PWM with period = 256 clock cycles

**Clock Source Selection:**
- TCCR1B bit 2:0 = 001: clkI/O / 1 (no prescaling, ~31kHz at 16MHz)
- TCCR2B bit 2:0 = 001: clkI/O / 1 (no prescaling, ~31kHz at 16MHz)

### Port/GPIO Control

**PORTD Mapping (Arduino Pins 0-7):**
```
Bit 5 → Arduino Pin 5 (Phase C low-side)
Bit 4 → Arduino Pin 4 (Phase B low-side)
Bit 3 → Arduino Pin 3 (Phase A low-side)
```

**PORTB Mapping (Arduino Pins 8-13):**
```
Bit 3 → Arduino Pin 11 (Timer2 OC2A - Phase A high PWM)
Bit 2 → Arduino Pin 10 (Timer1 OC1B - Phase C high PWM)
Bit 1 → Arduino Pin 9 (Timer1 OC1A - Phase B high PWM)
```

### Power Consumption Considerations

**Maximum Current Calculation:**
- Each MOSFET gate driven by IR2101 at 31kHz, 5V swing: ~5-10mA per driver
- Motor phase current at 12V: 5-10A typical (motor dependent)
- Control circuit: ~100mA total
- **Supply requirement**: 10-20A capable for typical 100W+ BLDC motor

### Thermal Considerations

**MOSFET Power Dissipation:**
- Each MOSFET dissipates I²R (conduction) + Pdrive
- 06N03LA RDS(on) ≈ 3-4 mΩ at full current
- At 5A per phase: ~75mW per MOSFET
- High-side MOSFETs may need small heatsink

**IR2101 Thermal Rating:**
- Typical operation at 50-70°C (well within 100°C max)
- Bootstrap capacitor stress only during high-speed operation

---

## Troubleshooting

### Common Issues and Solutions

| Issue | Likely Cause | Solution |
|-------|--------------|----------|
| Motor won't start | No BEMF signal, bad connection | Check all phase connections, verify 12V supply |
| Motor starts then stops | Comparator debounce too aggressive | Reduce debounce loop count in ISR |
| Motor vibrates | Timing issue, missed commutation | Check BEMF sensing network, verify resistor values |
| Motor not responding to speed control | PWM not working | Check Arduino pins 9, 10, 11 with meter |
| Inconsistent speed | Loose connections | Reflow all breadboard connections |
| Motor speed won't go below ~50% | PWM_MIN_DUTY too low | Increase PWM_MIN_DUTY to 75-100 |

### Debugging with Oscilloscope

**Recommended Test Points:**

1. **Phase Output**: Measure gate voltage at IR2101 output
   - Should see ~12V square wave at motor commutation frequency
   - Frequency proportional to motor speed

2. **BEMF Signal**: Measure Arduino pin 7 (Phase A BEMF)
   - Should see ~4-10V triangular/sawtooth waveform during operation
   - Frequency = motor RPM × (poles/2) / 60

3. **PWM Signal**: Measure Arduino pin 9, 10, or 11
   - Should see 31kHz square wave
   - Duty cycle varies with speed control (50-255/256)

4. **Neutral Point**: Measure Arduino pin 6
   - Should be stable at ~6V DC
   - Minimal AC ripple

### Code Modifications for Debugging

**Add serial debugging:**
```cpp
#include <SoftwareSerial.h>
// Monitor motor_speed and bldc_step via serial
```

**Test specific commutation steps:**
```cpp
// Modify bldc_move() to test each commutation pattern
// Comment out ISR to use manual step testing
```

---

## References

### Datasheets Required

- **Arduino UNO**: ATmega328P datasheet (Atmel)
- **Gate Driver**: IR2101 datasheet (Infineon)
- **MOSFET**: 06N03LA datasheet (ON Semiconductor)
- **Microcontroller**: ATmega328P datasheet (sections 27-31 for timers and analog comparator)

### Useful Reading

- "Brushless DC Motors" - Application Note by Microchip
- "Digital Commutation of Three-Phase Motors" - Motor Control Application
- IR2101 Bootstrap Supply Design - Infineon Application Note

### Project Attribution

This implementation is based on principles from simple-circuit.com with modifications for enhanced documentation and educational clarity.

### License

This project is provided as-is with NO WARRANTY for educational and hobbyist purposes.

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-06-03 | Initial comprehensive documentation |

---

## Contributing

For improvements, bug reports, or enhancements:
1. Test your changes thoroughly
2. Document any circuit modifications
3. Provide scope plots for new features
4. Submit detailed explanation of improvements

---

## Support

For issues or questions about this project, refer to the [Troubleshooting](#troubleshooting) section or consult the component datasheets mentioned in [References](#references).

**Last Updated**: June 3, 2026
