# 24 V → 10 V LDO Regulator Board

A small two-layer linear regulator board that takes a 24 V input on a screw terminal and delivers a fixed 10 V output on a second screw terminal. Built in KiCad 8, fabricated, and hand-assembled during my sophomore year in ECE at UT Austin.

The circuit is deliberately simple. The point of the project was to carry one board through the entire hardware flow — part selection from a manufacturer datasheet, schematic capture, custom symbol and footprint creation, layout, design review with more experienced peers, gerber export, fabrication, assembly, and bring-up — rather than to build something electrically clever.

![Assembled board](docs/images/assembled-board.png)

---

## Board at a glance

| | |
|---|---|
| Input | 24 V, 2-pin screw terminal |
| Output | 10 V, 2-pin screw terminal |
| Regulator | Nisshinbo Micro Devices **R1501** series LDO, TO-252-5 |
| Layers | 2 (F.Cu, B.Cu) |
| Board size | 44.04 × 19.04 mm |
| Board thickness | 1.6 mm |
| Vias | 4 |
| Mounting | 4 × 2.2 mm corner holes |
| Routing | Copper pours for +24 V and GND; 0.2 mm traces for +10 V |

Every number above is read directly out of `hardware/kicad/24-10Vboard.kicad_pcb` and `hardware/gerbers/24-10Vboard-job.gbrjob`.

---

## Why the R1501

I needed a regulator that could survive a 24 V rail and still source a useful amount of current at 10 V. The [Nisshinbo R1501 series](https://www.nisshinbo-microdevices.co.jp/en/products/ldo-linear-regulator/spec/?product=r1501) fit:

- Input range covering 24 V
- Output voltage available at 10 V
- 1 A class output current
- Built-in thermal shutdown, current limiting, and short-circuit protection
- Available in TO-252-5 (DPAK), a surface-mount package with a large thermal tab I could hand-solder

The part on the board is marked `R1501JxxxB-T1-FE1` in the schematic; the symbol I drew is `R1501SxxxB-E2-FE`. Both are R1501-family orderable variants, and reconciling that naming is one of the loose ends noted below.

KiCad had no symbol or footprint for this part, so I drew the symbol myself into a personal library (`sebastian_library`) straight from the datasheet pinout — Vdd, GND, GND, CE, Vout — and used the stock `Package_TO_SOT_SMD:TO-252-5_TabPin3` footprint.

**Datasheet:** [R1501 series (r1501-ea.pdf)](https://www.nisshinbo-microdevices.co.jp/en/pdf/datasheet/r1501-ea.pdf) — hosted by the manufacturer. I was not able to mirror a copy into this repo; the link is the authoritative source.

---

## Two things peers taught me

### 1. Copper pours instead of traces

My first layout routed the input and output rails as ordinary traces. A teammate with more layout experience pointed out that for a power board this is the wrong instinct: a trace has to be sized for its current, and a thin one both drops voltage and heats up. On a board this small there is no reason not to hand the whole rail as much copper as the outline allows.

The final layout leans on pours instead:

- **F.Cu** carries a `+24V` zone covering the input half of the board
- **B.Cu** carries a `GND` zone covering the whole board
- Both zones use 0.5 mm clearance, 0.25 mm minimum thickness, and 0.5 mm thermal relief gap
- Four vias stitch the pours between layers

This also gave the regulator's thermal tab a large copper area to dump heat into, which is what a linear regulator dropping 14 V actually needs. At 14 V of drop, every 100 mA of load turns into 1.4 W of heat inside that package — the copper pour *is* the heatsink.

What I did **not** do, and should have, is extend the same reasoning to the output. The `+10V` net is still routed the old way: four 0.2 mm segments running from the regulator's Vout pin past the output capacitor to the terminal. The 24 V input side gets both a pour and short 0.2 mm connecting segments; the 10 V output side gets traces only. That is the inconsistency a second review would have caught — see the notes at the end.

![KiCad layout showing the copper pours](docs/images/pcb-layout-kicad.png)

The red region is the +24 V pour on the front layer; the blue is the ground pour on the back.

### 2. Why the IC needs capacitors right next to its pins

I originally understood decoupling capacitors as "something you put on a power rail." What I did not understand was that where you put them is the entire point.

The explanation that made it click: the loop between the capacitor and the IC pin has inductance, and inductance is what resists a sudden change in current. When the regulator's load steps, the current has to come from somewhere immediately, faster than the 24 V supply and its wiring can respond. A capacitor a few millimetres from the pin can supply that current through a tiny loop. The same capacitor 30 mm away, on the far side of the board, cannot, because the loop inductance between them dominates. The capacitor stops being a capacitor and starts being an inductor at the frequencies that matter.

For an LDO specifically there is a second reason: the output capacitor is part of the regulator's feedback loop. It is not optional bulk storage — the control loop's stability depends on its capacitance and its ESR, which is why datasheets specify a range rather than a minimum.

So the layout places C2 (input) directly adjacent to the Vdd pin and C1 (output) directly adjacent to Vout, both with the shortest possible return path into the ground pour underneath.

---

## Repository layout

```
hardware/
  kicad/       schematic, PCB, and project file (KiCad 8)
  gerbers/     RS-274X gerbers + job file, as sent to the fab
docs/
  images/      layout screenshot, 3D render, assembled board photo
```

### Opening the KiCad project

The schematic references two personal libraries that are **not** included here:

- `sebastian_library:R1501SxxxB-E2-FE` — the symbol I drew for the regulator
- `scale:Screw_Terminal_2P` — the screw terminal footprint

KiCad will report these as unresolved and show the affected parts as placeholders. The PCB file itself is complete and self-contained — footprint geometry is embedded in `.kicad_pcb`, so the layout, the pours, and the gerbers all render correctly regardless. Packaging those libraries into the repo is on the list below.

---

## What I would do differently

Writing this up a year later, the gaps are easy to see:

- **The output rail is under-coppered.** The `+24V` input has a pour behind it; the `+10V` output is carried by 0.2 mm traces alone. A 0.2 mm trace in 1 oz copper is nowhere near sized for a regulator rated in the amps, so the output rail is what actually limits safe load current on this board. Pouring the output half of F.Cu as a `+10V` zone would have been the same edit I already made on the input side, and I simply did not carry the lesson across the IC.
- **No thermal analysis.** I never calculated the actual junction temperature at a given load current, which is the one calculation this board genuinely needed. A 24 V → 10 V linear drop is 58% of the input power turned into heat, and the safe operating current is set by the copper area.

None of these stopped the board from working. They are the difference between a board that works and a design that someone else could pick up and build.

---

## Note on the 3D render

![3D render](docs/images/3d-render.png)

The render shows tall blue aluminium electrolytic cans, while the assembled board has small yellow radial capacitors. This is a cosmetic mismatch only: KiCad attaches a default 3D model to each footprint, and the models here are the stock `Capacitor_THT.3dshapes/C_Radial_D5.0mm_H11.0mm_P2.00mm.wrl` and `C_Radial_D6.3mm_H11.0mm_P2.50mm.wrl`, which are drawn as electrolytic cans. The footprint — the pad positions, hole sizes, and lead pitch that actually get manufactured — is the same regardless of which picture the renderer attaches to it. The parts fit because their lead pitch and body diameter match the footprint, which is what matters.
---

*Sophomore-year personal project, ECE @ UT Austin. Designed in KiCad 8, fabricated externally, hand-assembled and bench-verified.*
