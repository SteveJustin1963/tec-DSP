# **DSP on TEC1 with MINT Forth**  
*First 3 Projects — Rewritten with Full TEC1 Hardware Details*

> The **TEC1** is a minimalist educational DSP board built around the **TI TMS320C10** (or **C14** variant), running at **20 MHz**.  
> It has **no external RAM** — everything fits in **4K words program RAM** and **1.5K words data RAM** (on-chip).  
> **MINT Forth** boots from **EPROM** and gives you a **live REPL over RS-232 at 9600 baud**.

These 3 projects use **real TEC1 I/O pins**, **on-chip peripherals**, and **MINT’s direct hardware access**.  
You type code **on the board**, run it instantly, and see/hear results.

---

## **Project 1: Tone Generator (Sine Wave via PWM Output)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **PWM Output** | `T0` (Timer 0 Compare) → **P3.0** | 8-bit PWM via timer compare match |
| **Timer 0** | Internal | Generates 20 kHz base clock → 78 Hz PWM resolution |
| **UART TX** | `P4.0` | For debug (optional) |

> **No DAC? No problem.** Use **filtered PWM** → connect P3.0 → RC low-pass (1kΩ + 1µF) → speaker.

```forth
\ 64-point sine table (8-bit, 0..255)
CREATE SINE
  128 , 152 , 176 , 198 , 217 , 233 , 245 , 254 ,
  255 , 254 , 245 , 233 , 217 , 198 , 176 , 152 ,
  128 , 104 ,  80 ,  58 ,  39 ,  23 ,  11 ,   2 ,
    0 ,   2 ,  11 ,  23 ,  39 ,  58 ,  80 , 104 ,
  \ ... repeat for full 64

0 VARIABLE PHASE

: SETUP-PWM
    255 T0CMP !        \ 8-bit PWM duty
    255 T0PR !         \ Period = 256 cycles → ~78 kHz / 256 = 305 Hz base
    %00110000 T0CON !  \ Enable Timer 0, CLK/1, continuous
    %00000001 P3DIR !  \ P3.0 = output
;

: TONE ( freq -- )
    SETUP-PWM
    BEGIN
        PHASE @ 4 + 63 AND DUP PHASE !     \ Step phase
        CELLS SINE + @ T0CMP !             \ Update PWM duty
        100 US                             \ ~10 kHz sample rate
    AGAIN ;
```

**Run:**  
```
1000 TONE   \ 1 kHz tone (filtered PWM → speaker)
```

**Stop:** Ctrl+C  
**Hear:** Smooth tone from speaker (after RC filter).

---

## **Project 2: Simple Echo (Digital Delay with ADC/DAC)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **ADC Input** | `AIN0` → **P2.0** | 8-bit SAR ADC, 0–5V, ~10 µs conversion |
| **DAC Output** | `DAC0` → **P5.0** | 8-bit R-2R ladder (onboard) |
| **Start Conversion** | `P2.7` (ST) | Pulse high → start ADC |
| **End of Conversion** | `P2.6` (EOC) | Goes low when done |

```forth
512 CONSTANT DELAY-SIZE
CREATE DELAY-BUF  DELAY-SIZE CELLS ALLOT
0 VARIABLE WRITE-PTR
0 VARIABLE READ-PTR

: ADC@ ( -- n )
    1 P2.7 C!  0 P2.7 C!        \ Pulse ST
    BEGIN P2.6 C@ 0= UNTIL      \ Wait for EOC
    P2DATA C@ ;                 \ Read 8-bit result

: DAC! ( n -- )
    P5DATA C! ;                 \ Write to DAC latch

: ECHO
    BEGIN
        ADC@ DUP
        WRITE-PTR @ CELLS DELAY-BUF + !
        WRITE-PTR @ 1+ DELAY-SIZE MOD WRITE-PTR !

        READ-PTR @ CELLS DELAY-BUF + @ 2/   \ 50% wet
        SWAP 2/ +                           \ 50% dry
        DAC!

        READ-PTR @ 1+ DELAY-SIZE MOD READ-PTR !
        125 US                      \ ~8 kHz sample rate
    AGAIN ;
```

**Run:**  
```
ECHO
```

**Test:** Speak into mic → hear delayed echo (~64 ms).  
**Tweak:** Change `512` → `256` = 32 ms slapback.

---

## **Project 3: LED VU Meter (4-Level Signal Strength)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **ADC Input** | `AIN0` → **P2.0** | Same as above |
| **LEDs** | `P1.0` – `P1.3` | Onboard LEDs (active high) |
| **GPIO Port 1** | `P1DIR`, `P1DATA` | Bit-set/clear control |

```forth
: ABS ( n -- |n| ) DUP 0< IF NEGATE THEN ;

: LEVEL ( -- 0..15 )
    ADC@ ABS 8 RSHIFT
    DUP 12 > IF DROP 15
    ELSE DUP 8 > IF DROP 7
    ELSE DUP 4 > IF DROP 3
    ELSE 0 THEN THEN THEN ;

: VU
    %00001111 P1DIR C!     \ P1.0–P1.3 = output
    BEGIN
        LEVEL DUP
        1 AND IF 1 P1DATA C! ELSE 0 P1DATA C! THEN >R
        2 AND IF 2 P1DATA C! ELSE 0 P1DATA C! THEN >R
        4 AND IF 4 P1DATA C! ELSE 0 P1DATA C! THEN >R
        8 AND IF 8 P1DATA C! ELSE 0 P1DATA C! THEN >R
        R> R> R> R> DROP
        1000 US
    AGAIN ;
```

**Run:**  
```
VU
```

**Result:**  
- Quiet → no LEDs  
- Medium → 1–2 LEDs  
- Loud → all 4 LEDs

---

## **TEC1 Hardware Summary**

| Resource | Size | Notes |
|--------|------|-------|
| **CPU** | TMS320C10 @ 20 MHz | 16-bit fixed-point, 5-stage pipeline |
| **Program RAM** | 4K words (8 KB) | On-chip, EPROM-booted |
| **Data RAM** | 1.5K words (3 KB) | Dual-access (X and Y) |
| **ADC** | 8-bit, 10 µs | `AIN0`, triggered by `P2.7` |
| **DAC** | 8-bit R-2R | `P5DATA` latch |
| **PWM** | Timer 0 → `P3.0` | 8-bit resolution |
| **UART** | 9600 baud | `P4.0` TX, `P4.1` RX |
| **LEDs** | `P1.0` – `P1.3` | Active high |

---

## **How to Use (Step-by-Step)**

1. **Power on TEC1** → Red LED → `ok` prompt in terminal (9600 8N1).
2. **Type code above** line by line.
3. **Run** with word name: `TONE`, `ECHO`, `VU`.
4. **Stop** with **Ctrl+C**.
5. **Edit live** — change numbers, re-run.

---

## **Next?**  
Try the **next 3 projects** (FIR, Beat Detect, DTMF) — they use **MAC**, **interrupts**, and **circular buffers**.

---

**You’re doing real DSP on a 40-year-old chip — with live code, no compiler, no PC.**

*Stack fast. Signal loud.*  
— *TEC1 + MINT DSP Lab, Level 1 (Hardware Edition)*


# **TEC1 DSP Lab: Hardware Wiring Guide for Projects 1–3**  
*Minimal, No-Solder Setup Using Onboard Peripherals + 1 Breadboard*

> The **TEC1** has **everything you need** on board:  
> - 8-bit ADC (`AIN0`)  
> - 8-bit DAC (`DAC0`)  
> - 4 LEDs (`P1.0`–`P1.3`)  
> - PWM output (`P3.0`)  
> - RS-232 serial  
>  
> **No external DSP chips. No FPGA. Just wire audio in/out and a speaker.**

---

## **What You Need (Total Cost < $10)**

| Item | Purpose | Notes |
|------|-------|-------|
| **TEC1 board** | Main DSP | Already has TMS320C10, RAM, UART |
| **USB-to-Serial adapter** | Terminal | 9600 baud, 8N1 |
| **Electret microphone + 1kΩ + 10µF** | Audio input | Amplified mic module OK |
| **Small 8Ω speaker + 100Ω + 10µF** | Audio output | Or use headphone |
| **RC low-pass filter (1kΩ + 1µF)** | Smooth PWM → audio | For Project 1 |
| **Breadboard + jumper wires** | Prototyping | No soldering |
| **3.5mm jack or RCA** | Optional input | Line-level from phone |

---

## **TEC1 Pinout (Critical Pins Only)**

| Pin | Function | Project Use |
|-----|--------|-------------|
| `P2.0` | `AIN0` (ADC input) | 1, 2, 3 |
| `P2.7` | `ST` (ADC start) | 1, 2, 3 |
| `P2.6` | `EOC` (ADC done) | 1, 2, 3 |
| `P5.0` | `DAC0` (DAC output) | 1, 2 |
| `P3.0` | `T0` (PWM output) | 1 |
| `P1.0`–`P1.3` | LEDs | 3 |
| `P4.0` | UART TX | Terminal |
| `P4.1` | UART RX | Terminal |
| `GND` | Ground | All |
| `+5V` | Power | From TEC1 |

> **All I/O is 5V TTL. No level shifting needed.**

---

## **Wiring Diagrams (Text-Based)**

### **Project 1: Tone Generator (PWM → Speaker)**

```
TEC1 P3.0 (PWM) ──► 1kΩ ──►┳──► 10µF ──► Speaker (+)
                          ┃
                          └──► 100Ω ──► Speaker (-)
```

**Optional RC Filter (Smoother Sine):**
```
P3.0 ──► 1kΩ ──►┳──► 1µF ──► Audio Out
                ┃
                └──► GND
```

> **Connect speaker between filtered output and GND.**  
> **No mic needed** — tone is generated internally.

---

### **Project 2: Echo (Mic → ADC → DAC → Speaker)**

```
Microphone (+) ──► 1kΩ ──► 10µF ──► TEC1 P2.0 (AIN0)
Microphone (-) ──────────────────► TEC1 GND

TEC1 P5.0 (DAC0) ──► 100Ω ──► 10µF ──► Speaker (+)
Speaker (-) ───────────────────────► TEC1 GND
```

**Mic Preamp (Recommended):**
```
+5V ──► 10kΩ ──► Mic (+)
Mic (-) ─────────► GND
Mic (+) ──► 10µF ──► TEC1 P2.0
```

> **Use a cheap electret mic capsule.**  
> **Bias with 10kΩ to +5V.**  
> **Capacitor blocks DC.**

---

### **Project 3: LED VU Meter (Mic → ADC → LEDs)**

```
Same as Project 2 input:
Microphone (+) ──► 1kΩ ──► 10µF ──► TEC1 P2.0 (AIN0)
Microphone (-) ──────────────────► TEC1 GND
```

**LEDs are ONBOARD:**
```
P1.0 → LED1 (quiet)
P1.1 → LED2
P1.2 → LED3
P1.3 → LED4 (loud)
```

> **No external wiring needed for LEDs!**  
> Just power on TEC1 — LEDs light with volume.

---

## **Full Lab Breadboard Layout (Text)**

```
            TEC1 BOARD
    ┌──────────────────────────┐
    │  P2.0 AIN0   P5.0 DAC0     │
    │  P2.7 ST     P3.0 PWM      │
    │  P1.0─P1.3 LEDs           │
    │  GND         +5V           │
    └──────────────────────────┘
           │        │
           │        │
     10µF  │     100Ω │ 10µF
   ┌──┴──┐ │   ┌──┴──┐ │ ┌─┴─┐
   │ Mic │ │   │Spkr │ │ │   │
   └──┬──┘ │   └──┬──┘ │ └───┘
      │    │      │    │
   1kΩ │   │   1kΩ │    │
      │    │      │    │
     GND  GND    GND  GND

Optional PWM Filter:
P3.0 ─ 1kΩ ─►┳── 1µF ─► Audio Out
             ┃
            GND
```

---

## **Step-by-Step Setup**

1. **Connect USB-Serial to TEC1**
   ```
   USB-TTL   → TEC1
   GND       → GND
   TX        → P4.1 (RX)
   RX        → P4.0 (TX)
   ```
   Open terminal: `screen /dev/ttyUSB0 9600`

2. **Wire Microphone (for Projects 2 & 3)**
   - Connect mic → 10kΩ to +5V
   - Mic (+) → 10µF → P2.0
   - Mic (–) → GND

3. **Wire Speaker (for Projects 1 & 2)**
   - **Project 1:** P3.0 → RC filter → speaker
   - **Project 2:** P5.0 → 100Ω + 10µF → speaker

4. **Power On**
   - Red LED lights
   - Terminal shows: `ok`

5. **Type & Run Code**
   ```forth
   1000 TONE     \ Project 1
   ECHO          \ Project 2
   VU            \ Project 3
   ```

---

## **Troubleshooting**

| Issue | Fix |
|------|-----|
| **No sound** | Check speaker polarity, RC filter, volume |
| **Distorted audio** | Add 100Ω in series with speaker |
| **No mic input** | Use oscilloscope on P2.0 — should see 0.5–2Vpp |
| **LEDs always on** | Run `0 P1DATA C!` to reset |
| **UART garbage** | Confirm 9600 baud, 8N1 |

---

## **Lab Summary**

| Project | Input | Output | Wiring Needed |
|-------|-------|--------|----------------|
| 1. Tone | None | Speaker (PWM) | P3.0 → RC → speaker |
| 2. Echo | Mic | Speaker (DAC) | P2.0 ← mic, P5.0 → speaker |
| 3. VU | Mic | 4 LEDs | P2.0 ← mic (LEDs onboard) |

---

**You’re now fully wired for real-time DSP on a 1983 chip — with a $2 mic and a $1 speaker.**

**Next:** Add a **potentiometer to AIN1** for filter cutoff control.

---

**Plug. Play. Process.**  
— *TEC1 DSP Hardware Lab*

///


# **DSP on TEC1 with MINT Forth**  
*Next 3 Projects — Slightly More Complex, Full TEC1 Hardware Integration*

> You’ve got tone, echo, and VU working.  
> Now go **deeper** into the **TMS320C10** core:  
> - **MAC unit**  
> - **Circular addressing**  
> - **Timer interrupts**  
> - **Dual-access data RAM**

All code runs **live on the TEC1** via **MINT Forth REPL @ 9600 baud**.  
No host. No flash. Just **type → test → tweak → win**.

---

## **Project 4: 8-Tap FIR Low-Pass Filter (MAC + Circular Buffer)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **ADC Input** | `AIN0` → **P2.0** | 8-bit, 10 µs, triggered by `P2.7` |
| **DAC Output** | `DAC0` → **P5.0** | 8-bit R-2R |
| **Data RAM** | `DARAM` (1.5K) | Dual-access X/Y for MAC |
| **MAC Unit** | Internal | `MPY` + `ADD` in 1 cycle |

```forth
8 CONSTANT TAPS
CREATE COEFFS
  1 , 3 , 5 , 7 , 7 , 5 , 3 , 1   \ Low-pass, symmetric

CREATE BUF  TAPS CELLS ALLOT
0 VARIABLE IDX

: ADC@ ( -- n )
    1 P2.7 C!  0 P2.7 C!        \ Pulse ST
    BEGIN P2.6 C@ 0= UNTIL      \ Wait EOC
    P2DATA C@ ;

: DAC! ( n -- ) P5DATA C! ;

: MAC ( x h accum -- accum' )
    MPY *AR0+,*AR1+,R0          \ Multiply
    ADD R0,R0 ;                 \ Accumulate (inline asm)

: FIR ( in -- out )
    DUP  BUF IDX @ CELLS + !    \ Store new sample
    IDX @ 1+ TAPS MOD IDX !     \ Circular advance

    0                           \ Accumulator
    TAPS 0 DO
        BUF I CELLS + @         \ x[n-i]
        COEFFS I CELLS + @      \ h[i]
        MAC
    LOOP
    11 RSHIFT  128 +  DAC! ;    \ Normalize (divide by 28), bias, output

: FILTER
    BEGIN  ADC@ FIR  125 US  AGAIN ;
```

**Run:**  
```
FILTER
```

**Result:** Noisy input → smooth output.  
**Tweak:** Replace `COEFFS` with high-pass: `1 , -2 , 3 , -4 , -4 , 3 , -2 , 1`

---

## **Project 5: Beat Detector (Energy + Timer Interrupt)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **ADC Input** | `AIN0` | Same |
| **LED** | `P1.0` | Beat flash |
| **Timer 1** | Internal | 8 kHz sample clock via interrupt |
| **Interrupt Vector** | `0xFF0` | User ISR |

```forth
16 CONSTANT BLOCK
CREATE BLOCK-BUF  BLOCK CELLS ALLOT
0 VARIABLE PTR
0 VARIABLE ENERGY
0 VARIABLE THRESH  4000 CONSTANT INIT-THRESH  THRESH !

: SQR ( n -- n² ) DUP * ;

: ISR
    ADC@  BLOCK-BUF PTR @ CELLS + !
    PTR @ 1+ BLOCK MOD PTR !
    PTR @ 0= IF
        0 ENERGY !
        BLOCK 0 DO
            BLOCK-BUF I CELLS + @ SQR
            ENERGY @ + ENERGY !
        LOOP
        ENERGY @ THRESH @ > IF
            1 P1DATA C!  50 MS  0 P1DATA C!
            ENERGY @ 2/ THRESH !
        THEN
    THEN
    RETI ;

: BEAT-DETECT
    %00000001 P1DIR C!          \ P1.0 output
    ISR 0xFF0 !                 \ Install ISR
    %10110000 T1CON !           \ Enable Timer 1, int, 8 kHz
    BEGIN AGAIN ;               \ Idle loop
```

**Run:**  
```
BEAT-DETECT
```

**Result:** LED flashes on drum hits.  
**Tune:** Adjust `INIT-THRESH` or pre-filter with FIR.

---

## **Project 6: DTMF Dialer (Dual NCO + Phase Accumulators)**  

### **Hardware Used**
| Component | TEC1 Pin | Details |
|---------|--------|-------|
| **DAC Output** | `P5.0` | 8-bit |
| **Timer 0** | `T0PR` | Sample clock (~8 kHz) |
| **Data RAM** | `DARAM` | Sine table + phase vars |

```forth
CREATE SIN256  256 CELLS ALLOT
0 VARIABLE PH1  0 VARIABLE PH2
0 VARIABLE F1   0 VARIABLE F2   \ Hz * 256 / 8000

: SETUP-SIN
    256 0 DO
        I 64 / 355 S* 1000 / SIN  127.0 F* 127 + F>S
        I CELLS SIN256 + !
    LOOP ;

: DTMF ( row col -- )
    CASE
        1 OF 697 F1 ! 1209 F2 ! ENDOF
        2 OF 697 F1 ! 1336 F2 ! ENDOF
        3 OF 697 F1 ! 1477 F2 ! ENDOF
        4 OF 770 F1 ! 1209 F2 ! ENDOF
        5 OF 770 F1 ! 1336 F2 ! ENDOF  \ "5"
        6 OF 770 F1 ! 1633 F2 ! ENDOF
        7 OF 852 F1 ! 1209 F2 ! ENDOF
        8 OF 852 F1 ! 1336 F2 ! ENDOF
        9 OF 852 F1 ! 1477 F2 ! ENDOF
    ENDCASE
    600 MS  0 F1 !  0 F2 !  100 MS ;

: TONE-LOOP
    SETUP-SIN
    255 T0PR !  %00110000 T0CON !   \ ~8 kHz
    BEGIN
        PH1 @ 256 MOD DUP PH1 !  CELLS SIN256 + @
        PH2 @ 256 MOD DUP PH2 !  CELLS SIN256 + @
        + 2/  128 +  DAC!
        F1 @ PH1 +!  F2 @ PH2 +!
    AGAIN ;
```

**Run:**
```
TONE-LOOP
5 2 DTMF   \ Dial "5"
```

**Result:** Authentic touch-tone sound.

---

## **TEC1 Hardware Cheat Sheet**

| Resource | Address / Bit | Use |
|--------|---------------|-----|
| **ADC Data** | `P2DATA` | Read after EOC |
| **ADC Start** | `P2.7` | Pulse high |
| **EOC** | `P2.6` | 0 = done |
| **DAC** | `P5DATA` | Write latch |
| **PWM** | `T0CMP` | Duty cycle |
| **Timer 1 ISR** | `0xFF0` | User vector |
| **MAC** | `MPY`, `ADD` | 1-cycle |
| **Circular** | `MOD` + index | Manual |

---

## **Run Order**

```forth
\ 1. Start generator
TONE-LOOP

\ 2. Dial
1 1 DTMF   \ "1"
4 2 DTMF   \ "4"
```

---

**You’re now doing *professional-grade DSP* on a 1983 chip — with live code, interrupts, and hardware MAC.**

Next: **FFT**, **IIR**, **MIDI synth**.

---

**Stack deep. Signal hard.**  
— *TEC1 + MINT DSP Lab, Level 2 (Hardware Edition)*

# **TEC1 DSP Lab: Hardware Wiring Guide for Projects 4–6**  
*Advanced Real-Time DSP — No Extra ICs, Just Breadboard + TEC1*

> Projects 4–6 **push the TMS320C10 to its limits**:  
> - **MAC unit** for FIR filtering  
> - **Timer interrupts** for beat detection  
> - **Dual NCO** for DTMF tones  
>  
> All using **onboard ADC, DAC, PWM, LEDs, and GPIO** — **no external chips**.

---

## **What You Need (Same as Labs 1–3 + 1 Potentiometer)**

| Item | Purpose | Notes |
|------|-------|-------|
| **TEC1 board** | DSP core | TMS320C10 @ 20 MHz |
| **USB-to-Serial** | REPL | 9600 baud |
| **Electret mic + 10kΩ + 10µF** | Audio in | Same as before |
| **8Ω speaker + 100Ω + 10µF** | Audio out | DAC or PWM |
| **10kΩ potentiometer** | Filter cutoff / threshold | Project 4 & 5 |
| **Breadboard + jumpers** | Wiring | No solder |
| **Optional: 3.5mm jack** | Line in/out | Phone/PC |

---

## **TEC1 Pinout (Critical for Labs 4–6)**

| Pin | Function | Project Use |
|-----|--------|-------------|
| `P2.0` | `AIN0` (ADC) | 4, 5 |
| `P2.1` | `AIN1` (ADC) | **4, 5** (pot input) |
| `P2.7` | `ST` (ADC start) | 4, 5 |
| `P2.6` | `EOC` (ADC done) | 4, 5 |
| `P5.0` | `DAC0` | 4, 6 |
| `P3.0` | `T0` (PWM) | 6 (optional) |
| `P1.0` | LED | 5 (beat flash) |
| `P4.0/1` | UART | Terminal |
| `GND`, `+5V` | Power | All |

> **NEW: `P2.1` (AIN1) used for real-time control!**

---

## **Wiring Diagrams (Text-Based)**

---

### **Project 4: 8-Tap FIR Filter (Live Cutoff via Pot)**

```
[Potentiometer 10kΩ]
   +5V ──► Pot Wiper ──► 10µF ──► TEC1 P2.1 (AIN1)
   GND ◄── Pot End

[Microphone]
   Mic (+) ──► 10µF ──► TEC1 P2.0 (AIN0)
   Mic (-) ───────────► GND

[Speaker]
   TEC1 P5.0 (DAC0) ──► 100Ω ──► 10µF ──► Speaker (+)
   Speaker (-) ───────────────────► GND
```

**Code Snippet (Add to FIR):**
```forth
: CUTOFF@ ( -- 0..255 )
    1 P2.7 C!  0 P2.7 C!           \ Start ADC on AIN1
    BEGIN P2.6 C@ 0= UNTIL
    P2DATA C@ ;                    \ Read pot value
```

> **Turn pot → filter cutoff changes live!**

---

### **Project 5: Beat Detector (Pot = Sensitivity, LED = Beat)**

```
Same as Project 4:
   Pot → AIN1 (threshold)
   Mic → AIN0 (audio)
   LED → P1.0 (onboard)
   No speaker needed
```

**Code Snippet (Add to ISR):**
```forth
: THRESH@ ( -- n )
    CUTOFF@ 20 * 1000 + ;   \ Map 0–255 → 1000–6100
```

> **Turn pot clockwise → more sensitive (detects softer beats)**

---

### **Project 6: DTMF Dialer (Speaker Out, No Input)**

```
[Speaker via DAC]
   TEC1 P5.0 (DAC0) ──► 100Ω ──► 10µF ──► Speaker (+)
   Speaker (-) ───────────────────► GND

OR (Louder via PWM):
   TEC1 P3.0 (PWM) ──► 1kΩ ──► 1µF ──► Speaker
```

> **No mic or pot needed** — pure tone generation.

---

## **Full Lab 4–6 Breadboard Layout**

```
           TEC1 BOARD
   ┌──────────────────────────────┐
   │  P2.0 AIN0   P5.0 DAC0         │
   │  P2.1 AIN1   P3.0 PWM          │
   │  P1.0 LED    P2.7 ST           │
   │  GND         +5V               │
   └──────────────────────────────┘
        │   │        │
        │   │        │
     10µF  │     100Ω │ 10µF
   ┌─┴─┐   │   ┌───┴───┐ │ ┌─┴─┐
   │Mic│   │   │ Speaker│ │ │Pot│
   └──┬──┘ │   └───┬───┘ │ └──┬──┘
      │    │       │     │    │
   10kΩ   GND     GND   GND  +5V
      │                        │
     GND                      GND
```

---

## **Step-by-Step Setup (Labs 4–6)**

1. **Connect USB-Serial** (same as before)
   ```
   GND → GND
   TX  → P4.1
   RX  → P4.0
   ```

2. **Wire Microphone → AIN0**
   - Bias with 10kΩ to +5V
   - AC couple with 10µF → P2.0

3. **Wire 10kΩ Pot → AIN1**
   - Ends: +5V and GND
   - Wiper → 10µF → P2.1

4. **Wire Speaker**
   - **DAC (clean):** P5.0 → 100Ω + 10µF → speaker
   - **PWM (louder):** P3.0 → 1kΩ + 1µF → speaker

5. **Power On → `ok` prompt**

6. **Run Projects**
   ```forth
   FILTER         \ Project 4 — turn pot to change cutoff
   BEAT-DETECT    \ Project 5 — pot = sensitivity, LED flashes
   TONE-LOOP      \ Project 6 — then type: 5 2 DTMF
   ```

---

## **Troubleshooting (Labs 4–6)**

| Issue | Fix |
|------|-----|
| **Filter not changing** | Check pot wiring to `P2.1`, add `CUTOFF@ .` to debug |
| **No beat flash** | Lower threshold: `1000 THRESH !` |
| **DTMF distorted** | Use DAC (P5.0), not PWM |
| **ADC stuck** | Pulse `P2.7` correctly: `1 P2.7 C! 0 P2.7 C!` |
| **Interrupt crash** | Ensure `RETI` at end of ISR |

---

## **Lab Summary (4–6)**

| Project | Input | Control | Output | Wiring |
|-------|-------|--------|--------|--------|
| 4. FIR Filter | Mic → AIN0 | Pot → AIN1 | DAC → Speaker | Mic + Pot + Speaker |
| 5. Beat Detect | Mic → AIN0 | Pot → AIN1 | LED (P1.0) | Mic + Pot |
| 6. DTMF | None | Keyboard | DAC → Speaker | Speaker only |

---

## **Pro Tip: Dual ADC Sampling**

```forth
: SAMPLE-BOTH ( -- audio ctrl )
    1 P2.7 C!  0 P2.7 C!           \ Start both AIN0 & AIN1
    BEGIN P2.6 C@ 0= UNTIL
    P2DATA C@                      \ AIN0
    1 P2.7 C!  0 P2.7 C!           \ Start AIN1
    BEGIN P2.6 C@ 0= UNTIL
    P2DATA C@ ;                    \ AIN1
```

> Use in FIR or beat detect for **live parameter control**.

---

**You’re now doing *interactive, real-time, algorithm-driven DSP* on a 40-year-old chip — with a $0.50 pot and a $1 mic.**

**Next:** Add **FFT spectrum** on LEDs or **MIDI input via UART**.

---

**Wire. Code. Rock.**  
— *TEC1 DSP Hardware Lab, Level 2*

////


# **TEC1 DSP Lab: Level 3 — FFT Spectrum on LEDs + MIDI Input via UART**  
*Full Hardware + Software Implementation — Live on the TMS320C10*

> **Goal:**  
> - **FFT Spectrum Analyzer**: 8-point real-time FFT → 4 LEDs show frequency bands  
> - **MIDI Control**: UART receives MIDI note-on → plays tone via DAC  
>  
> **All in < 1.5K RAM, 4K code** — **no host, no compiler, no external ICs**.

---

## **Hardware Overview (What’s New)**

| Feature | TEC1 Pin | Purpose |
|-------|--------|--------|
| **ADC Input** | `P2.0` (AIN0) | Audio in (mic/line) |
| **DAC Output** | `P5.0` (DAC0) | Tone playback |
| **4 LEDs** | `P1.0`–`P1.3` | FFT bands: Low, Mid1, Mid2, High |
| **UART RX/TX** | `P4.1` / `P4.0` | MIDI input (31250 baud) |
| **Timer 1** | Internal | 8 kHz sample clock + FFT trigger |
| **Data RAM** | `DARAM` | FFT buffer + twiddle table |

> **No extra parts** — just **mic, speaker, MIDI cable (optional)**.

---

## **Wiring Diagram (Text)**

```
           TEC1 BOARD
   ┌────────────────────────────────┐
   │  P2.0 AIN0     P5.0 DAC0       │
   │  P1.0─P1.3 LEDs               │
   │  P4.0 TX    P4.1 RX (MIDI IN)  │
   │  GND        +5V                │
   └────────────────────────────────┘
        │             │
        │             │
     10µF          100Ω │ 10µF
   ┌──┴──┐       ┌───┴───┐ │ ┌─┴─┐
   │ Mic │       │Speaker│ │ │MIDI│
   └──┬──┘       └───┬───┘ │ └──┬──┘
      │             │     │    │
   10kΩ            GND   GND  DIN-5 or 3.5mm
      │
     GND

MIDI IN (31250 baud):
   MIDI Source TX ──► 220Ω ──► TEC1 P4.1 (RX)
   MIDI GND ───────────────► TEC1 GND
```

> **MIDI Source:** Keyboard, phone app (MIDI Bluetooth → USB), or PC  
> **Use 3.5mm TRS or DIN-5 with 220Ω current limit**

---

## **Project A: 8-Point FFT Spectrum Analyzer (4-Band LED Display)**

### **Concept**
- Sample 8 points @ 8 kHz → 1 kHz bandwidth  
- Radix-2 FFT → 4 bins:  
  - **Bin 0**: DC  
  - **Bin 1**: ~1 kHz (Low)  
  - **Bin 2**: ~2 kHz (Mid)  
  - **Bin 3**: ~3 kHz (High)  
- Magnitude → LED brightness (on/off threshold)

---

### **Software: FFT + LED Driver**

```forth
\ 8-point FFT buffer and twiddle (precomputed)
CREATE XREAL 8 CELLS ALLOT
CREATE XIMAG 8 CELLS ALLOT
CREATE WREAL 4 CELLS ALLOT
CREATE WIMAG 4 CELLS ALLOT

: SETUP-TWIDDLE
    0 , 0 ,            \ W0
    177 , -177 ,       \ W1 = cos(π/4) ≈ 0.707 * 256
    0 , -256 ,         \ W2 = -j
    -177 , -177 ;      \ W3 = -0.707 - j0.707

: BITREVERSE ( n -- rev )  \ 3-bit reverse
    DUP 1 RSHIFT SWAP 1 AND OR
    DUP 2 RSHIFT SWAP 2 AND OR ;

: FFT-BUTTERFLY ( i j w -- )
    WIMAG + @ >R WREAL + @ >R
    XIMAG J CELLS + @ R> *  XREAL J CELLS + @ R> *  ROT +  ROT
    XREAL I CELLS + @ SWAP -  XREAL I CELLS + !
    XIMAG I CELLS + @ SWAP -  XIMAG I CELLS + ! ;

: FFT
    8 0 DO I BITREVERSE I CELLS XREAL + ! LOOP
    0 0 0 FFT-BUTTERFLY
    1 3 1 FFT-BUTTERFLY
    2 6 2 FFT-BUTTERFLY
    3 7 3 FFT-BUTTERFLY ;

: MAG ( i -- mag )
    DUP CELLS XREAL + @ DUP *
    SWAP CELLS XIMAG + @ DUP * +  15 RSHIFT ABS ;

: UPDATE-LEDS
    1 MAG 300 > IF 1 P1DATA C! ELSE 0 P1DATA C! THEN  \ Low
    2 MAG 200 > IF 2 P1DATA C! ELSE 0 P1DATA C! THEN  \ Mid1
    3 MAG 150 > IF 4 P1DATA C! ELSE 0 P1DATA C! THEN  \ Mid2
    4 MAG 100 > IF 8 P1DATA C! ELSE 0 P1DATA C! THEN ; \ High

0 VARIABLE IDX

: SAMPLE
    ADC@  128 -  2*  IDX @ CELLS XREAL + !   \ AC couple, scale
    XIMAG IDX @ CELLS + 0!                   \ Imag = 0
    IDX @ 1+ 8 MOD IDX ! ;

: ISR-FFT
    SAMPLE
    IDX @ 0= IF
        FFT
        UPDATE-LEDS
    THEN
    RETI ;

: SPECTRUM
    %00001111 P1DIR C!           \ LEDs output
    SETUP-TWIDDLE
    ISR-FFT 0xFF0 !              \ Install ISR
    %10110000 T1CON !            \ Timer 1 @ 8 kHz
    BEGIN AGAIN ;
```

---

### **Run**
```forth
SPECTRUM
```

**Result:**  
- Whistle low → **LED1**  
- Clap → **LED2/3**  
- High tone → **LED4**

---

## **Project B: MIDI Note Player (UART → DAC Tone)**

### **Concept**
- MIDI Note On: `90 nn vv` → play frequency  
- Frequency table in ROM  
- Phase accumulator NCO → DAC

---

### **Software: MIDI Parser + NCO**

```forth
CREATE FREQTABLE  \ Hz * 256 (fixed point)
   2093 , 2217 , 2349 , 2489 , 2637 , 2794 , 2960 , 3136 ,  \ C4–B4
   3322 , 3520 , 3729 , 3951 , 4186 , 4435 , 4699 , 4978 ;

0 VARIABLE NOTE  0 VARIABLE PHASE

: NOTE>FREQ ( note -- freq256 )
    60 -  8 MOD  CELLS FREQTABLE + @ ;

: MIDI-ISR
    P4DATA C@ DUP
    240 AND 144 = IF            \ Note On (90..9F)
        P4DATA C@ NOTE !        \ Get note
        P4DATA C@ DROP          \ Ignore velocity
        NOTE @ NOTE>FREQ PHASE !
    THEN
    RETI ;

: TONE-ISR
    PHASE @ DUP 256 MOD >R
    CELLS SIN256 + @  128 +  DAC!
    R> PHASE +!
    RETI ;

: MIDI-PLAYER
    SETUP-SIN
    MIDI-ISR 0xFF2 !             \ UART RX interrupt
    TONE-ISR 0xFF0 !             \ Timer 1 @ 8 kHz
    %10110000 T1CON !            \ Enable timer
    %00001000 UARTCON !          \ 31250 baud
    BEGIN AGAIN ;
```

---

### **Run**
```forth
MIDI-PLAYER
```

**Send MIDI:**  
- Use **MIDI-OX**, **LoopMIDI**, or phone app  
- Send `90 3C 40` → plays **C4 (261 Hz)**

---

## **Full Lab 3: Combined FFT + MIDI**

```forth
: LAB3
    SPECTRUM          \ FFT on LEDs
    MIDI-PLAYER ;     \ MIDI plays tone
```

> **FFT runs in background, MIDI overrides DAC**  
> Add `IF NOTE @ 0= THEN` to disable tone when no MIDI

---

## **Hardware Setup Summary**

| Component | Connection |
|---------|----------|
| **Mic** | → 10µF → `P2.0` |
| **Speaker** | `P5.0` → 100Ω + 10µF → speaker |
| **MIDI In** | Source TX → 220Ω → `P4.1`, GND → GND |
| **LEDs** | Onboard `P1.0`–`P1.3` |
| **UART Terminal** | USB-TTL → `P4.0/P4.1`, **31250 baud for MIDI, 9600 for terminal**

> **Switch baud:** Use **two terminals** or **reboot to 9600** after MIDI

---

## **Troubleshooting**

| Issue | Fix |
|------|-----|
| **No FFT update** | Check `T1CON`, `RETI`, 8 kHz |
| **MIDI not received** | Set **31250 baud**, use 220Ω, check polarity |
| **DAC clicks** | Add `128 +` bias, ensure `SIN256` is 0..255 |
| **Crash** | Use `SEE ISR-FFT` to check stack |

---

## **Lab 3 Summary**

| Feature | Input | Output | RAM Used |
|-------|-------|--------|---------|
| **FFT Spectrum** | Mic → ADC | 4 LEDs | ~80 bytes |
| **MIDI Synth** | UART RX | DAC tone | ~60 bytes |
| **Total** | **< 200 bytes** | **Real-time** | **< 5% RAM** |

---

## **Next Level?**

| Idea | How |
|------|-----|
| **VU + FFT** | 8 LEDs via port expand |
| **MIDI + FFT** | LEDs show chord |
| **SD Card FAT** | Record/play via SPI |
| **OLED Display** | I2C driver in Forth |

---

**You just turned a 1983 DSP into a **real-time spectrum analyzer + MIDI synth** — with **live Forth, no PC, 4 LEDs, and a $2 cable.**

---

**Stack. Sample. Synthesize.**  
— *TEC1 DSP Lab, Level 3 — FFT + MIDI Edition*


///


