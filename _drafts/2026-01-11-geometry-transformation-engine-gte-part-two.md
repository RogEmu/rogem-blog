---
layout: post
title: Geometry Transformation Engine (GTE) Part 2 In-Depth Functioning
date: 2026-01-11 21:37 +0100
categories:
  - Emulation
tags:
  - technical
  - component
  - hardware
  - internals
---
## First part of this article
In the first part of this article we went over the basis of what is the GTE for the Playstation.
If you did not read it yet, we encourage you to do it before continuing.

Now we will dwelve into more detailled example of what the GTE does and how it works, before finishing with a quick note on why it is essential to implement it correctly.

## How we model the GTE in our emulator
In our emulator, the GTE is implemented as a **COP2** device with two register banks:

- **Control registers**: `m_ctrlReg[0..31]` (COP2C)
- **Data registers**: `m_dataReg[0..31]` (COP2D)

This matches how the PS1 CPU accesses the GTE: it does not use memory-mapped IO, it uses COP2 opcodes to move values in/out and execute commands.

### Quick “mental map” of the registers we touch the most

> This is **not** a full register table — it’s the subset that matters for understanding the code below.

#### Control (COP2C) highlights
- Rotation matrix base (RT): used by `RTPS/RTPT` and also by `OP` (diagonal elements)
- Translation vector TR: `TRX/TRY/TRZ` (we extract them via `extractTranslation(5)` in RTPS)
- Projection parameters:
  - `OFX` = ctrl[24]
  - `OFY` = ctrl[25]
  - `H`   = ctrl[26]
  - `DQA` = ctrl[27]
  - `DQB` = ctrl[28]
- Color pipeline:
  - `RFC/GFC/BFC` live around ctrl[21..23] in classic docs
  - Our code also uses ctrl[21..23] as interpolation targets in `INTPL` (see below)

#### Data (COP2D) highlights
- `IR1/IR2/IR3` = data[9..11] (intermediate vector)
- `SXY0/SXY1/SXY2` are stored as packed `(SY << 16) | SX` in data[12..14]
- `SZ3` used in RTPS is stored into data[19]
- `MAC0` often ends up in data[24] in our code (ex: `NCLIP`, `AVSZ`)
- `MAC1/2/3` are written in data[25..27]
- Color FIFO head is packed into data[20] by `pushColorFIFO`

---

## Fixed-point arithmetic in practice (what “sf” really does)
The GTE is fixed-point: the PS1 stores many matrix/vector values as 16-bit signed with **12 fractional bits** (commonly called **S.3.12** style).

In our implementation, most math is performed as `int64_t` then optionally shifted.

- `sf = 1` → shift right by 12 after MAC accumulation (`>> 12`)
- `sf = 0` → keep full precision (no 12-bit downshift)

You’ll see this pattern everywhere:
```cpp
int64_t mac = (...) >> (sf * 12);
```

### Why it matters
If your test vectors are “in GTE scale” (S.12 fractional), then:
- `sf=1` tends to bring results back to “normal IR scale”
- `sf=0` can overflow earlier or saturate differently
The RTPS/RTPT docs explicitly mention saturation differences depending on sf and how IR flags behave.

---

## Coordinate Transformation (RTPS / RTPT)
One of the most important tasks of the GTE is transforming a 3D point from model space into screen space.
The transformation pipeline is roughly:
```
Model space → World space → View space → Screen space
```

This is implemented in the GTE using matrix-vector multiplication and perspective division.
The most common instruction for this is:
```
RTPS (Rotate, Translate, Perspective Single)
```

It performs:
1. Rotation of a vertex using the rotation matrix
2. Translation using the translation vector
3. Perspective projection to screen coordinates

The output is written into the screen registers `SX`, `SY` and `SZ`, which can then be sent to the GPU to draw polygons.

### Our RTPS dataflow (schematic)
Below is the exact *shape* of what our code does:
```
Input vertex Vn (from data regs)               Projection params (ctrl regs)
┌──────────────────────────────┐              ┌────────────────────────────┐
│ Vx, Vy, Vz (int16)           │              │ OFX, OFY, H, DQA, DQB      │
└───────────────┬──────────────┘              └─────────────┬──────────────┘
                │                                           │
                v                                           v
     ┌──────────────────────────┐              ┌─────────────────────────┐
     │ MAC1..3 = TR*4096 + R*V  │              │ projectScale = H / SZ3  │
     │ then >> (sf*12)          │              │ clamped to 0..0x1FFFF   │
     └──────────────┬───────────┘              └────────────┬────────────┘
                    │                                       │
                    v                                       v
       ┌────────────────────────────┐         ┌──────────────────────────────┐
       │ IR1..3 = clamp(MAC1..3)    │         │ SX2,SY2 = (scale*IR + OF)/16 │
       │ SZ3   = clamp(MAC3 >> ...) │         │ packed into SXY2 (data[14])  │
       └──────────────┬─────────────┘         └─────────────┬────────────────┘
                      │                                     │
                      v                                     v
                 ┌─────────┐                      ┌─────────────────┐
                 │ MAC regs│                      │ DQA/DQB depth   │
                 └─────────┘                      │ push: data[24]  │
                                                  │ IR0-ish: data[8]│
                                                  └─────────────────┘

```

### RTPT in our emulator
`RTPT` is simply “RTPS executed 3 times”:
```cpp
for (uint8_t i = 0; i < 3; i++)
    executeRTPS(opcode, i);
```

This matches the conceptual meaning: transform V0, V1, V2 and update the screen/depth FIFOs.

### Example (with “made-up but realistic” values)
Assume:
- `sf = 1`
- rotation = identity
- translation = (0, 0, 0)
- `H = 0x100` (256)
- `OFX = OFY = 0`

Vertex:
- `V0 = (4096, 0, 8192)`  
  (meaning X=1.0, Y=0.0, Z=2.0 if using 12 fractional bits)

Then:
- `MAC3 ≈ 8192 >> 12 = 2`
- `SZ3` becomes small → in our code if `SZ3 <= H/2` we clamp and set a flag, and use max scale.

This is exactly why fixed-point + threshold logic matters: “near camera” and “behind camera” paths are very sensitive.

---

## Normal Clipping (NCLIP)
`NCLIP` is used to help decide if a triangle is front-facing or back-facing in screen space.

Our code:
- reads `SXY0` = data[12], `SXY1` = data[13], `SXY2` = data[14]
- extracts each `(sx, sy)` from the packed words
- computes:
```
MAC0 = (SX0*SY1 + SX1*SY2 + SX2*SY0) - (SX0*SY2 + SX1*SY0 + SX2*SY1)
```

and stores it into `MAC0` (we write it to `data[24]`).
This matches official training material and references.

### Schematic
```
SXY0, SXY1, SXY2
  │     │     │
  v     v     v
Extract (sx0,sy0) (sx1,sy1) (sx2,sy2)
  │
  v
MAC0 = oriented area / 2 (signed)
  │
  v
data[24] = MAC0
```
If `MAC0` is:
- positive → one winding
- negative → the other winding

---

## The “vector math” helpers: OP and SQR

### OP (Outer Product / Cross Product)
In PSX docs this opcode is sometimes called “Outer Product”, but it’s effectively the cross product in practice.

In our emulator:
- Vector A = `(IR1, IR2, IR3)`
- Vector B = `(D1, D2, D3)` where:
  - `D1 = RT11`
  - `D2 = RT22`
  - `D3 = RT33`
  (diagonal of rotation matrix)

Then:
```
MAC1 = IR3*D2 - IR2*D3
MAC2 = IR1*D3 - IR3*D1
MAC3 = IR2*D1 - IR1*D2
```

Then we apply `>> (sf*12)`, store into `MAC1..3` and clamp into `IR1..3`.
This matches the spec note about “misusing RT diagonal as a vector”.

#### Example
Let:
- `IR = (1000, 2000, 3000)`
- `D = (1, 2, 3)` (from RT11/RT22/RT33)
- `sf = 0` (no shift)

Then:
- `MAC1 = 3000*2 - 2000*3 = 6000 - 6000 = 0`
- `MAC2 = 1000*3 - 3000*1 = 3000 - 3000 = 0`
- `MAC3 = 2000*1 - 1000*2 = 2000 - 2000 = 0`

So the result is the zero vector (because A and D were aligned in a specific way).
This is a great unit-test shape: it’s easy to verify, and it stresses “do we read D from the correct matrix slots?”.

### SQR (Square Vector)
Our `SQR`:
- reads IR1..3
- squares each component
- applies `>> (sf*12)`
- stores into MAC1..3 and clamps into IR1..3

This instruction is often used for vector magnitude-ish workflows.

#### Example
Let:
- `IR = (200, -300, 400)`
- `sf = 0`

Then:
- `MAC1 = 200*200 = 40000` → likely clamps into IR range
- `MAC2 = (-300)*(-300) = 90000` → clamps
- `MAC3 = 400*400 = 160000` → clamps

This is exactly the kind of instruction where saturation behavior defines whether geometry “explodes” or stays stable.


---

## Average Z (AVSZ3 / AVSZ4)
The GPU uses an “ordering table” approach, and the GTE provides helpers to compute an average depth:
- `AVSZ3` averages 3 Z values
- `AVSZ4` averages 4 Z values

In our emulator we share code:
1. Sum SZ values from the SZ FIFO (we iterate starting at SZ3 and going backward)
2. Multiply by `ZSF3` or `ZSF4` (read from a control register)
3. Shift down by 12
4. Clamp to `[0 .. 0xFFFF]`
5. Store into OTZ (lower halfword of data[7])
6. Store MAC0 for inspection/debug

This aligns with the idea of “scaled average depth”, even if real hardware has a lot of nuance around FIFO ordering and flags.

---

## General matrix math (MVMVA)
`MVMVA` is the workhorse “matrix * vector + translation” command.

In our code, it is driven by opcode flags:
- `mx` selects which matrix to use
- `v` selects which vector source to use
- `cv` selects which translation base to apply
- `sf` controls shift (`>> 12` or not)
- `lm` changes the lower clamp bound for IR (either `-0x8000` or `0`)

We:
- build the matrix (special case when `mx == 3`, otherwise extract it)
- fetch the selected vector
- fetch the translation vector
- compute MAC1..3
- clamp into IR1..3 with a lower bound that depends on `lm`

This instruction is often used for:
- transforming normals
- transforming light directions
- “small pipelines” inside other operations

---

## Lighting and Color: our “executeNColor” pipeline
This is the most “emulator-specific” part of this article: it’s not just “what the GTE does”, it’s how **we structured our implementation** to cover a *family* of opcodes with one shared pipeline function.

The GTE has multiple color-related commands:
- NCS / NCT
- NCCS / NCCT
- NCDS / NCDT
- CC
- CDP

The PSX-SPX docs group these into “color calculation commands” with shared concepts:
- background color BK
- color matrix LCM
- far color FC
- interpolation factor IR0
- and the RGB FIFO

### Our pipeline switches
Our function:
```cpp
executeNColor(normal, sf, isNormal, color, depth)
```
interprets flags like this:
- `isNormal == true`  
  → do a first stage “LLM * normal” to produce IR
- `color == true`  
  → apply a color multiplication stage (think NCCx/CC)
- `depth == true`  
  → apply a depth-cue interpolation stage (think NCDx/CDP)

### Pipeline schematic
```
(Option A) Normal transform stage (if isNormal)
  normal (V0/V1/V2) --[LLM]--> MAC1..3 --> clamp --> IR1..3

(Always) Background + ColorMatrix stage
  BK + [LCM]*IR --> MAC1..3 --> clamp --> IR1..3

(Option B) Color and/or Depth stage (if color||depth)
  IR scaled/colored --> MAC1..3
  if depth: interpolate with IR0 (lerp toward "Far Color")
  shift >> (sf*12) --> clamp --> IR1..3

(Final) Pack to RGB and push FIFO
  RGB = clamp(MAC >> 4) into [0..255]
  push into color FIFO, update data[20]
```

### FIFO behavior in our code
We model a 3-entry FIFO:
```cpp
Rgbc m_colorFIFO[3]; // RGB0, RGB1, RGB2
```

When we push:
- RGB0 <- RGB1
- RGB1 <- RGB2
- RGB2 <- new

And we pack the newest into `data[20]` as:
```
(CODE << 24) | (R << 16) | (G << 8) | (B)
```

---

## Interpolation helpers (DPCS / DPCT / DCPL / INTPL / GPF / GPL)
These are “color math building blocks” in the GTE.

In our implementation:
- `DPCS` builds a MAC vector from the current `RGBC` and calls `INTPL`
- `DPCT` applies that to 3 consecutive RGB values (loop)
- `DCPL` multiplies `RGBC` components by IR and then calls `INTPL`
- `INTPL` performs a 3-channel interpolation using:
  - current MAC values
  - target colors (ctrl[21..23] in our code)
  - interpolation factor `data[8]`
  then pushes the result to the color FIFO

Separately:
- `GPF` and `GPL` use IR * IR0-ish scaling with optional base addition, then push to FIFO

This is the kind of thing that makes the GTE feel like a “graphics DSP” rather than a pure transform unit.


---


## Why The GTE Matters For Emulation
The GTE is responsible for almost all of the 3D math for the PlayStation.

If the GTE is inaccurate, you will see:
- Warped geometry
- Broken lighting
- Incorrect depth
- Flickering or exploding polygons

A correct emulator must reproduce:
- Instruction timing and pipeline semantics
- Register saturation rules (especially IR and RGB paths)
- Fixed-point precision and `sf` behavior
- Flag behavior (FLAG is not “optional”, games *do* depend on it)

The GTE is one of the hardest parts of a PlayStation emulator to implement correctly, but also one of the most rewarding.


## Implementation notes (based on our current code)
This section is here because emulator blogs are most useful when they document *real decisions and real gotchas*.

### 1) NCS decode currently calls NCCS
In `decodeAndExecute`:
```cpp
case GTEFunction::NCS: {
    bool sf = (opcode >> 19) & 0x1;
    executeNCCS(sf);
    break;
}
```

This means NCS (normal color single) is currently routed to the NCCS path (normal color color single).
If you see unexpected extra “color multiplication behavior” in games, this is a prime suspect.

### 2) Saturation is subtle (and games care)
No$PSX docs emphasize that:
- MAC registers themselves are generally not saturated
- IR/RGB saturations set FLAG bits
- some commands treat lm differently (RTPS/RTPT behave as if lm=0)

When debugging rendering bugs, always verify:
- where you clamp (MAC vs IR vs RGB)
- whether you set flags on “pre-shift” or “post-shift” values

---

## References

1. [https://psx-spx.consoledev.net/geometrytransformationenginegte/](https://psx-spx.consoledev.net/geometrytransformationenginegte/)
2. [https://problemkaputt.de/psx-spx.htm](https://problemkaputt.de/psx-spx.htm)
3. [https://psx.arthus.net/sdk/Psy-Q/DOCS/TRAINING/FALL96/gte.pdf](https://psx.arthus.net/sdk/Psy-Q/DOCS/TRAINING/FALL96/gte.pdf)