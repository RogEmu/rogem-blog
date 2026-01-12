---
layout: post
title: Geometry Transformation Engine (GTE) Part 1 Overhaul
date: 2026-01-11 16:52 +0100
categories:
  - Emulation
tags:
  - technical
  - component
  - hardware
  - internals
---
## The GTE In Computers
In modern computers, 3D geometry processing is performed by the GPU using dedicated vector and matrix units.  
These units are responsible for transforming 3D coordinates into screen space, computing lighting, and preparing vertices for rasterization.

The original PlayStation does not have a programmable GPU in the modern sense. Instead, it uses a **fixed-function coprocessor** dedicated to geometry math: the **Geometry Transformation Engine (GTE)**.

The GTE is tightly coupled to the CPU and is designed to accelerate:
- 3D coordinate transformations
- Perspective projection
- Lighting and color computations
- Depth cueing
- Vector and matrix operations

Without the GTE, all of these operations would have to be executed using the main MIPS CPU, which would be far too slow for real-time 3D games.

## GTE On The PS1
The GTE is implemented as **Coprocessor 2 (COP2)** of the PlayStation’s R3000A CPU.

Unlike peripherals such as the GPU or SPU, the GTE is not accessed through memory-mapped registers.  
Instead, it is controlled using special CPU instructions (`COP2` opcodes) that directly operate on its internal registers.

The CPU sends vectors, matrices, and parameters into the GTE, executes a GTE instruction, and reads the results back a few cycles later.

The GTE is designed around a **pipeline**: several operations can be in flight at the same time, and many instructions have execution delays before their results become available.

## GTE Registers
The GTE exposes two groups of registers:

### Data Registers (DR)
These registers store:
- Input vectors
- Intermediate results
- Output screen coordinates and colors

They include registers such as:
- `VX0, VY0, VZ0` (input vertex)
- `SX, SY, SZ` (screen space coordinates)
- `RGB` (color output)
- `MAC` and `IR` (accumulators and interpolated values)

### Control Registers (CR)
These contain:
- Rotation matrices
- Translation vectors
- Lighting matrices
- Projection and depth parameters

Typical control registers include:
- Rotation matrix (R11…R33)
- Translation vector (TRX, TRY, TRZ)
- Light matrices
- Screen offset and projection plane distance

Together, these registers define how 3D data is transformed and lit.

## Second part of the article

Since the GTE is a big part of the Playstation, we decided to divide this article into two parts.

The first part you just read was an overhaul of what a GTE is and how it works. 
It will soon be followed by a second, bigger part with in-depth coverage of how it functions and how we implemented it.

## References

1. [https://psx-spx.consoledev.net/geometrytransformationenginegte/](https://psx-spx.consoledev.net/geometrytransformationenginegte/)
2. [https://problemkaputt.de/psx-spx.htm](https://problemkaputt.de/psx-spx.htm)
3. [https://psx.arthus.net/sdk/Psy-Q/DOCS/TRAINING/FALL96/gte.pdf](https://psx.arthus.net/sdk/Psy-Q/DOCS/TRAINING/FALL96/gte.pdf)
