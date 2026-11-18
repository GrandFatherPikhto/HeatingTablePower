# Reflow Oven & Heating Table Power Board (Dual Triac Switch)

*Читать на русском: [README_ru.md](README_ru.md)*

Hardware design (KiCad schematic and PCB layout) for the mains-side power board of the dual-channel reflow oven / heating table controller ([`../ControllerSTM32F103C8T6`](../ControllerSTM32F103C8T6)).

The board switches two resistive heaters from 220 V AC with **BTA12-600CRG** triacs. Each triac is driven through a **MOC3063M** zero-crossing optotriac, so the logic side is galvanically isolated from the mains.

---

## Key Features & Hardware Architecture

* **Two identical heater channels:** one hierarchical sheet (`channel.kicad_sch`) is placed twice, as `Channel 1` and `Channel 2`. Each channel has:
  * a BTA12-600CRG triac (`Q1` / `Q2`; 12 A RMS, 600 V, TO-220, insulated tab);
  * a MOC3063M optotriac (`U2` / `U3`) with a 330 Ω gate resistor (`R9` / `R12`, 1210);
  * an RC snubber across the triac: 39 Ω (`R10` / `R13`, axial) + 0.1 µF 630 V (`C7` / `C8`);
  * a heater output terminal (`J6` / `J7`).

  On the PCB, channel 2 is the channel 1 layout shifted 20 mm to the right.
* **Zero-cross switching:** the optotriacs turn the triacs on only near the mains zero crossing. This reduces inrush current and EMI. It also means the outputs suit on/off or burst-fire (whole half-cycle) control, not phase-angle dimming.
* **Logic input:** the control inputs come from `J4` and accept 3.3 V or 5 V logic. Each passes a 1 kΩ series resistor (`R3`, `R4`) into a Schmitt-trigger inverter `U1`, which must be a **74HCT14** (see note 5). Each input has a 10 kΩ pull-down (`R1`, `R2`), so a disconnected cable means "heater off".
* **Optotriac drive:** the LED of each MOC3063M is fed from `+5P` through 330 Ω (`R8`, `R11`) and is sunk by the `U1` output, giving ≈ 10–11 mA (the MOC3063M needs 5 mA max to trigger).
* **Status LEDs:** a second inverter of `U1` in each channel drives the channel LED, which is lit while the heater is on:
  * `D2` (purple, `R6`) — channel 1;
  * `D1` (blue, `R5`) — channel 2;
  * `D3` (red, `R7`) — logic power present.
* **Net ties `NT1` / `NT2`:** they split the thin optotriac branch (`UP_ARM`, 0.5 mm track) off the heavy heater net (`220V_UP`). The branch is therefore a separate net with its own width, and the mains-track rule in `Power.kicad_dru` does not apply to it.
* **Mains input protection:** varistor `RV1` (SIOV-S14K275, 275 V AC) and an across-the-line capacitor `C1` (0.1 µF 630 V). **There is no fuse on the board** — see the notes.
* **Logic supply filtering:** `C4` (47 µF 10 V), `C5` (10 µF) and `C6` (100 nF) at `J5`; `C2` (100 nF) and `C3` (10 µF) at `U1`.
* **Board:** 2 layers, ≈ 93 × 65 mm, 4× M3 mounting holes.

---

## Repository Structure

* `Power.kicad_pro` — KiCad project file.
* `Power.kicad_sch` — root schematic sheet. Its hierarchical sheets:
  * `signal.kicad_sch` (`Signal`) — `J4`, the input network, `U1` and the channel LEDs;
  * `power.kicad_sch` (`Power`) — `J5`, logic supply filtering and the power LED;
  * `220v_in.kicad_sch` (`220V`) — `J1`, `RV1`, `C1`;
  * `channel.kicad_sch` (`Channel 1`, `Channel 2`) — one heater channel.
* `Power.kicad_pcb` — PCB layout.
* `Power.kicad_dru` — Custom design rule: tracks of the mains nets (`*/220V*`) may have exposed copper (no solder mask) so they can be tinned for extra current capacity.
* `gerber/` — Gerber and drill files. They are dated 27.10 and belong to the previous, flat-schematic revision. Regenerate them before ordering.
* `out/` — BOM and netclass audit reports.

---

## Connector Pinouts

### J1 — Mains Input (`220V In`, Phoenix MKDS 1.5, 5.08 mm)

| Pin | Signal | Description |
| :---: | :--- | :--- |
| 1 | `/220V` | Goes to A1 of both triacs, the snubbers, `RV1` and `C1`. Connect **Line (L)** here. |
| 2 | `/220V_COMMON` | Goes straight to pin 2 of both heater terminals. Connect **Neutral (N)** here. |

### J4 — Heater Control Inputs (`Signal Input`, JST-XH 4-pin)

Pin-compatible with `J10 (Signal Out)` of the controller board (straight cable).

| Pin | Signal | Description |
| :---: | :--- | :--- |
| 1 | `GND` | Ground |
| 2 | `HEATER_IN_01` | Channel 1 control, active high (3.3 / 5 V logic) |
| 3 | `GND` | Ground |
| 4 | `HEATER_IN_02` | Channel 2 control, active high (3.3 / 5 V logic) |

### J5 — Logic Power Input (`+5V In`, JST-XH 2-pin)

| Pin | Signal | Description |
| :---: | :--- | :--- |
| 1 | `+5P` | +5 V supply for `U1`, the LEDs and the optotriac LEDs |
| 2 | `GND` | Ground (shared with the controller via `J4`) |

### J6 / J7 — Heater Outputs (`Heater Out`, Phoenix MKDS 1.5, 5.08 mm)

| Pin | Signal | Description |
| :---: | :--- | :--- |
| 1 | `220V_UP` | Switched output from the triac (A2) |
| 2 | `/220V_COMMON` | Heater common (Neutral) |

### Channel Map

The channels run straight through:

| Input (`J4`) | `U1` gates | Optotriac | Triac | Output | Status LED |
| :--- | :---: | :---: | :---: | :---: | :--- |
| `HEATER_IN_01` (pin 2) | 1→2, 13→12 | `U2` | `Q1` | `J6` | `D2` (purple) |
| `HEATER_IN_02` (pin 4) | 3→4, 11→10 | `U3` | `Q2` | `J7` | `D1` (blue) |

The unused inputs of `U1` (pins 5 and 9) are tied to `GND`.

---

## Assembly & Safety Notes

1. **Mains voltage.** All of the following are at mains potential: `J1`, `J6`, `J7`, the triacs, gate resistors `R9`/`R12`, the snubbers, `NT1`/`NT2`, `RV1`, `C1` and pins 4–6 of the optotriacs. Do not touch the board while it is plugged in.
2. **Fusing is external.** The board has no fuse. Install a fuse (or circuit breaker) in the Line wire **before** `J1`, rated for the total current of both heaters. The varistor `RV1` relies on that upstream fuse: when overloaded it fails short. Make sure the mains traces can carry the fuse rating.
3. **`C1` must be an X2 safety capacitor.** It sits directly across the line. X2 parts are also the safe choice for the snubber capacitors `C7`/`C8`. The footprints (10 mm pitch, MKS4-size) fit standard X2 film capacitors.
4. **Heatsinks.** A BTA12 dissipates roughly 1–1.5 W per ampere of heater current. Above ~2 A per channel, mount the triacs on a heatsink (the BTA tab is insulated, so both can share one).
5. **`U1` must be a 74HCT14.** The controller drives `J4` from a 74HC14 powered at 3.3 V, so a high level is about 3.3 V. The HCT input threshold is TTL-level (VIH = 2.0 V) and accepts it with margin. A plain 74HC14 at 5 V has its positive Schmitt threshold at up to ≈ 3.5 V and is not guaranteed to see 3.3 V as high.
6. **Fail-safe input.** The pull-downs `R1`/`R2` keep both heaters off when the cable is unplugged. On the controller side, the inputs of its `U6` are pulled down too, so the heaters also stay off while the STM32 is in reset or being flashed.
