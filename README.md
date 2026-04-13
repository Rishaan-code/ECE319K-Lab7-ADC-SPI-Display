# ECE319K Lab 7 — ADC Sensing and LCD Display System

## Overview
This project builds a real-time position sensing system on the 
MSPM0G3507. A slide potentiometer connected to PB18 gets sampled 
by a 12-bit ADC at 30 Hz using a hardware timer interrupt. The raw 
ADC value gets converted to a fixed-point distance measurement 
(0.000 to 2.000 cm) using a calibrated linear equation, then 
displayed on an ST7735 LCD over SPI. The system also plots a 
scrolling history of position values on the LCD screen.

This lab was the foundation for Lab 8, where the same ADC and 
Convert pipeline got extended into a full distributed UART 
communication system across two microcontrollers.

## How It Works

### ADC Sampling
The slide potentiometer outputs a voltage between 0 and 3.3V on 
pin PB18. ADC1 channel 5 samples this voltage and returns a 
12-bit result (0 to 4095). The ADC runs at 40-48 MHz with a 
divide-by-8 clock, software triggered, no hardware averaging.

### Calibration and Convert Function
Five physical measurements were taken at known potentiometer 
positions and recorded in Calibration.xls. A linear regression 
fit produced two constants k1 and k2 that map raw ADC values to 
distance in units of 0.001 cm:
Position = ((k1 * Data) >> 12) + k2
Position = ((1739 * Data) >> 12) + 148
The right shift by 12 keeps everything in integer arithmetic 
without floating point. This matters on the Cortex M0+ because 
floating point operations take roughly 100x longer than integer 
operations. FloatConvert was implemented alongside Convert 
specifically to benchmark this difference using SysTick timing.

### Fixed Point Display
OutFix takes the integer position value (0 to 2000) and formats 
it as X.XXX cm for the LCD. No floating point involved. The 
whole and fractional parts are split with integer division and 
modulo:

```c
whole = n / 1000;   // ones digit
part  = n % 1000;   // three decimal digits
printf("%u.%03u cm", whole, part);
```

### Interrupt Architecture
TimerG12 fires every 33.3ms (30 Hz) and sets a semaphore flag. 
The main loop spins waiting on the flag, clears it when set, 
then runs the Convert and OutFix pipeline. This keeps the ADC 
sampling precisely timed regardless of how long the LCD update 
takes.
TimerG12 ISR (30 Hz):        Main loop:
Sample ADC then Wait for flag
Set Flag = 1 then Clear flag
Return then Convert raw ADC to distance
then Display on LCD
then Plot scrolling graph
### Scrolling Plot
Every 15 samples (about 2 Hz) the system plots a point on the 
ST7735 LCD and advances the cursor, creating a scrolling 
position history graph. ST7735_PlotNextErase handles the 
scrolling behavior.

## Hardware Configuration
| Component | Detail |
|---|---|
| Microcontroller | TI MSPM0G3507 (80 MHz) |
| ADC | ADC1, 12-bit, channel 5, PB18 |
| Display | ST7735 LCD via SPI |
| Potentiometer | Slide pot, pin 2 to PB18 |
| Sampling rate | 30 Hz via TimerG12 |
| Timer period | 2,666,667 cycles (80 MHz / 30 Hz) |

## Calibration
Calibration was performed by measuring raw ADC output at 5 known 
physical positions across the slide potentiometer range. A linear 
fit through these points produced:
k1 = 1739
k2 = 148
Position (0.001 cm) = ((1739 * ADC) >> 12) + 148
Full range: ADC = 0 → Position ≈ 148 (0.148 cm)  
Full range: ADC = 4095 → Position ≈ 1881 (1.881 cm)

## Performance Measurements
Measured using SysTick cycle counter at 80 MHz:

| Operation | Time |
|---|---|
| ADCin() execution | ~1 µs |
| Convert() execution | < 1 µs (integer math) |
| FloatConvert() execution | ~100x slower than Convert |
| OutFix() execution | ~6 ms (LCD SPI transfer) |
| Sampling period | 33.3 ms (30 Hz) |

The FloatConvert vs Convert comparison was the most interesting 
result. The Cortex M0+ has no hardware floating point unit so 
every float operation gets emulated in software. The integer 
fixed-point approach is dramatically faster and good enough for 
this application.

## Files
| File | Description |
|---|---|
| `ADC1.c` | ADC init, sampling, Convert, FloatConvert |
| `Lab7Main.c` | Multiple test mains, final interrupt system |
| `Calibration.xls` | Raw calibration data and linear fit |

## Test Structure
The lab was built incrementally through six test mains:

- **main1** -- verified potentiometer voltage range using TExaS logic analyzer
- **main2** -- tested ADCinit and ADCin, measured execution time with SysTick
- **main3** -- tested Convert accuracy against physical measurements
- **main4** -- tested OutFix display timing on LCD
- **main5** -- final system with TimerG12 interrupt and scrolling plot
- **main6** -- Central Limit Theorem study using histogram averaging

## What I Learned
The biggest takeaway from this lab was understanding the real cost 
of floating point on embedded hardware. I implemented both a 
fixed-point and floating point version of the same conversion and 
measured the difference directly with the SysTick timer. Seeing 
that gap in actual clock cycles made the case for fixed-point 
arithmetic much more concrete than any textbook explanation could.

Getting the semaphore pattern right between the ISR and main loop 
was also a good exercise. The ISR just sets a flag and returns 
immediately. It does the minimum possible work. Main does the 
heavy lifting. That separation of responsibilities carried 
directly into Lab 8 where the same pattern scaled up to a full 
producer-consumer UART system.
