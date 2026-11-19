# Color-Mixer-Project

# 🎨 Automated RGB Color Mixer

Automated color-mixing line using **two Siemens S7-1200 PLCs (Main + AUX)**, a **Schneider Altivar ATV12 VFD** (Modbus RTU), **WinCC SCADA** and a **Schneider Magelis XBTGT2330 HMI**. Red, Green and Blue liquids are dispensed into a 1 L glass carried by a closed-loop conveyor, then mixed.

No flow meter and no proximity sensors: volume is **timer-based (1 s = 50 mL)** and position is **encoder-based (HSC)**.

## Hardware

| Component | Role |
|---|---|
| Main PLC, S7-1200 | Logic, sequencing, HMI/SCADA interface |
| AUX PLC, S7-1200 + CM1241 (RS485) | Physical I/O, HSC encoder, Modbus master |
| Altivar ATV12 | Conveyor motor speed and direction |
| Magelis XBTGT2330 | Local HMI |
| WinCC | SCADA screens, animation, alarms |
| Encoder | Position feedback (AUX `I0.0`) |

## Architecture

```
WinCC SCADA + Magelis HMI
          │
     MAIN PLC (logic)
          │  Profinet PUT/GET (10 Hz)
      AUX PLC + CM1241
       │            │
  Physical I/O   Modbus RTU ─► Altivar ATV12 ─► Conveyor motor
```

The S7-1200 CPU has no native RS485 port, so the AUX PLC with the CM1241 acts as the hardware and Modbus gateway.

## Process

`HOME → RED → GREEN → BLUE → MIXER → back HOME`

1. Operator enters mL, RGB, or picks a palette color, then confirms.
2. At home (Encoder = 0) the inputs are **locked** and the timer presets are calculated: `ms = mL × 1000 / 50`.
3. The interlock checks volume (0 < total ≤ 1000 mL) and tank levels, then allows the run.
4. At each station the conveyor stops (hardware interrupt), the valve opens until the timer finishes, then the conveyor moves on. Colors with 0 mL are skipped.
5. At the mixer: cylinder down (1 s), mix (5 s), cylinder up (1 s).
6. The conveyor reverses to home. Single-color batches skip the mixer.

## I/O (AUX PLC)

| Address | Signal |
|---|---|
| I0.0 / I0.1 / I0.2 / I0.3 | Encoder / Start / Reset / Stop (NC) |
| Q0.0 / Q0.1 / Q0.2 | Red / Green / Blue valve |
| Q0.3 / Q0.4 | Cylinder / Mixer |

## Communication

**Profinet (Main ⇄ AUX):** Main sends valves, cylinder, mixer, direction (`MW5`), encoder reset (`M32.0`), target position (`MD200`) and motor flags (`M204.1–3`). It reads the encoder (`ID1000`), buttons, `HIStop` (`M204.0`) and the VFD feedback.

**Modbus RTU (AUX ⇄ ATV12):** 9600 baud, even parity, slave 3, using `MB_COMM_LOAD` and `MB_MASTER`.

| Register | Function |
|---|---|
| 8501 (CMD) | Stop = `6`, Forward = `15`, Reverse = `2063` |
| 8502 (LFR) | Frequency setpoint (Hz × 10) |
| 3202 / 3204 / 3208 | Output frequency / current / voltage (read) |

**Precise stopping:** Main sends the next station position to `MD200`. When the HSC count equals it, **OB40** fires, stops the VFD over Modbus and sets `M204.0`. Main reads it as `HIStop`.

## Program Structure (Main PLC, SCL / Ladder)

| Block | Function |
|---|---|
| OB1 | Calls all FCs. Auto and manual FCs are selected by the Manual flag |
| FC1, FC2 | Communication and VFD feedback / SetFreq |
| FC3, FC4 | PE / NE edge functions |
| FC5–FC8 | SCADA / HMI input conversion (mL ⇄ RGB) |
| FC9 | Home logic: value snapshot, timer presets, encoder reset |
| FC10 | Timers (TONR) |
| FC11 | Timer → mL, tank levels, live color preview |
| FC12 | Interlock: run permission and safety checks |
| FC13 | Auto sequence: stations, mixer, return home |
| FC14 | Manual mode and protections |

**AUX PLC:** OB1 (`CTRL_HSC`, Profinet), OB40 (hardware interrupt stop), VFD FC (run command + Modbus), `VFD PARAMETERS` DB.

## Key Logic

- **Locked batch values:** `HMI.VR → "VR OK" → VR`. The copy to `VR` happens only at Encoder = 0.
- **TONR timers** pause instead of resetting, so volume stays correct after interruptions.
- **Tanks:** 5000 mL each. Warning below 1000 mL; start is blocked if the request exceeds the tank level.
- **Live color preview:** `R = 255 × red_mL / glass_mL` (same for G, B). Display only.

## Alarms

Volume = 0 · Volume > 1000 mL · Tank low · Insufficient tank · Tank empty / glass full / ColorFall (manual) · Hardware interrupt stop

## HMI & SCADA

Pages: Operation, RGB, Volume, Color palette, Alarms, Manual, VFD.
Conveyor animation is linked to the **Encoder (DINT)** (exact glass position), not the motor flags. Confirm (OK) buttons must create a rising edge to transfer the entered values.

## Calibration

Run the conveyor slowly at a fixed frequency, stop at each station, and read the HSC value (`ID1000`). Average 3 runs, then update FC13 (positions, HI zones) and the FC14 ColorFall ranges (position ± 50).

Example: Red 692 · Green 1098 · Blue 1521 · Mixer 1928

## Testing

PLCSIM simulates the **Main PLC only**. Modify logic inputs (Encoder, Marker flags, SCADA/HMI values) in a watch table. Keep Stop = TRUE (NC).

## Authors

_[Names]_ · _License_
