# ARM Pipelined Processor with Dynamic Branch Prediction

[![Architecture](https://img.shields.io/badge/ISA-ARM%20v7-blue)](https://developer.arm.com/architectures)
[![Language](https://img.shields.io/badge/HDL-Verilog-orange)](https://en.wikipedia.org/wiki/Verilog)
[![Testing](https://img.shields.io/badge/Verification-Cocotb-green)](https://www.cocotb.org/)
[![Platform](https://img.shields.io/badge/FPGA-Validated-success)](https://en.wikipedia.org/wiki/Field-programmable_gate_array)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A high-performance **5-stage pipelined ARM processor** with advanced hazard handling and dynamic branch prediction. Features a sophisticated branch predictor combining BTB, GHR, and PHT for minimizing control hazard penalties. Designed for educational purposes and FPGA implementation.

## Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
  - [Pipeline Stages](#pipeline-stages)
  - [Block Diagram](#block-diagram)
  - [PC Selection Priority](#pc-selection-priority)
- [Branch Prediction System](#branch-prediction-system)
  - [Branch Predictor Architecture](#branch-predictor-architecture)
  - [Branch Target Buffer (BTB)](#branch-target-buffer-btb)
  - [Global History Register (GHR)](#global-history-register-ghr)
  - [Pattern History Table (PHT)](#pattern-history-table-pht)
- [Hazard Handling](#hazard-handling)
  - [Data Hazards](#data-hazards)
  - [Control Hazards](#control-hazards)
- [Instruction Support](#instruction-support)
  - [Data Processing](#data-processing-instructions)
  - [Memory Operations](#memory-operations)
  - [Branch and Control Flow](#branch-and-control-flow)
- [Getting Started](#getting-started)
- [Testing and Verification](#testing-and-verification)
- [Technical Specifications](#technical-specifications)
- [Design Decisions](#design-decisions)
- [Future Enhancements](#future-enhancements)
- [Author](#author)
- [References](#references)

---

## Overview

This project implements a **5-stage pipelined ARM processor** based on the classic RISC pipeline architecture with advanced modifications for enhanced performance. The processor features:

- **Dynamic Branch Prediction**: 3-component predictor (BTB + GHR + PHT)
- **Comprehensive Hazard Handling**: Data forwarding, pipeline stalling, and flush mechanisms
- **Full Pipeline**: Fetch → Decode → Execute → Memory → Writeback stages
- **Educational Focus**: Clear implementation emphasizing computer architecture principles

**Key Characteristics:**
- 32-bit datapath with standard ARM instruction encoding
- Hardware-based branch prediction to minimize control hazard penalties
- Data forwarding to reduce pipeline stalls
- Validity tracking to prevent false hazard detection
- FPGA-validated design with comprehensive testbenches

**Use Cases:**
- **Computer Architecture Education**: Understanding pipelining and hazards
- **FPGA Prototyping**: Synthesizable processor core
- **Branch Prediction Research**: Studying predictor effectiveness
- **Digital Design Coursework**: Advanced processor implementation

---

## Features

### Pipeline Architecture
- **5-Stage Pipeline**: Fetch, Decode, Execute, Memory, Writeback
- **Single Clock Domain**: Synchronous design throughout
- **Harvard Architecture**: Separate instruction and data paths
- **Register File**: 32×32-bit registers with dual read ports, single write port
- **Negative Edge Write**: Register file writes on falling clock edge

### Hazard Resolution
- **Data Forwarding**:
  - Forward from Memory stage to Execute stage
  - Forward from Writeback stage to Execute stage
  - Validity checking to prevent false matches
- **Pipeline Stalling**: Load-use hazard detection (LDR stall)
- **Pipeline Flushing**: Branch misprediction recovery
- **Validity Signals**: Prevent hazards from invalid register addresses

### Branch Prediction
- **Branch Target Buffer (BTB)**: 3-entry fully associative cache
- **Global History Register (GHR)**: 3-bit shift register tracking branch patterns
- **Pattern History Table (PHT)**: 8-entry table with 1-bit saturating counters
- **Misprediction Recovery**: Automatic flush and PC correction

### Datapath Features
- **Barrel Shifter**: Integrated shifter before ALU B port
- **Immediate Extension**: Support for various immediate formats
- **PC Management**: PC+4 carried through pipeline for link operations
- **Flexible ALU**: Supports arithmetic, logic, and shift operations

---

## Architecture

### Pipeline Stages

```
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│ Fetch  │───>│ Decode │───>│Execute │───>│ Memory │───>│Writeback│
│  (F)   │    │  (D)   │    │  (E)   │    │  (M)   │    │  (W)   │
└────────┘    └────────┘    └────────┘    └────────┘    └────────┘
    │             │             │             │             │
    ▼             ▼             ▼             ▼             ▼
┌────────────────────────────────────────────────────────────────┐
│                    Pipeline Registers                          │
│  F/D Register │ D/E Register │ E/M Register │ M/W Register    │
└────────────────────────────────────────────────────────────────┘
```

**Stage Descriptions:**

| Stage | Operations | Key Signals |
|-------|-----------|-------------|
| **Fetch** | - Fetch instruction from memory<br>- Branch prediction (BTB + PHT)<br>- PC selection | PC, Instruction, BTA, BranchPredicted |
| **Decode** | - Instruction decode<br>- Register file read<br>- Immediate extension<br>- Control signal generation | RD1, RD2, ExtImm, RA1, RA2, WA3 |
| **Execute** | - ALU operation<br>- Shift operation<br>- Branch condition evaluation<br>- Forwarding logic | ALUResult, BranchTaken, ForwardA, ForwardB |
| **Memory** | - Data memory access (LDR/STR)<br>- Address calculation result | ReadData, ALUOut, MemWrite, MemRead |
| **Writeback** | - Register file write<br>- Result selection (ALU/Memory)<br>- PC update for branches | ResultW, RegWrite, WA3W |

### Block Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ARM Pipelined Processor Core                       │
└──────────────────────────────────────────────────────────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐         ┌─────────────────┐        ┌─────────────────┐
│Branch Predictor│        │    Datapath     │        │  Hazard Unit    │
│               │        │                 │        │                 │
│  ┌─────┐      │        │  ┌──────────┐  │        │  ┌──────────┐   │
│  │ BTB │      │        │  │   ALU    │  │        │  │ Forward  │   │
│  └─────┘      │        │  └──────────┘  │        │  │  Logic   │   │
│  ┌─────┐      │        │  ┌──────────┐  │        │  └──────────┘   │
│  │ GHR │─────>│        │  │ Shifter  │  │        │  ┌──────────┐   │
│  └─────┘      │        │  └──────────┘  │        │  │  Stall   │   │
│  ┌─────┐      │        │  ┌──────────┐  │        │  │  Logic   │   │
│  │ PHT │      │        │  │  RegFile │  │        │  └──────────┘   │
│  └─────┘      │        │  └──────────┘  │        │  ┌──────────┐   │
│               │        │  ┌──────────┐  │        │  │  Flush   │   │
└───────────────┘        │  │  Extend  │  │        │  │  Logic   │   │
                         │  └──────────┘  │        │  └──────────┘   │
                         └─────────────────┘        └─────────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐         ┌─────────────────┐        ┌─────────────────┐
│     PC        │         │ Instruction Mem │        │   Data Memory   │
│   Register    │         │                 │        │                 │
└───────────────┘         └─────────────────┘        └─────────────────┘
```

### PC Selection Priority

The processor implements sophisticated PC selection with priority ordering:

```
┌─────────────────────────────────────────────────────┐
│              PC Selection Multiplexer                │
└─────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PRIORITY 1  │  │  PRIORITY 2  │  │  PRIORITY 3  │
│              │  │              │  │              │
│  ALUResultE  │  │Carried PC+4  │  │ Predictor    │
│              │  │              │  │   Output     │
│  (Branch     │  │ (Mispredicted│  │              │
│   should be  │  │  branch not  │  │ BTA or PC+4  │
│   taken)     │  │  taken)      │  │              │
└──────────────┘  └──────────────┘  └──────────────┘

Condition:           Condition:        Default:
BranchTakenE=1   |   BranchTakenE=0  |  Use predictor
BranchPredE=0    |   BranchPredE=1   |  decision
```

**Priority Logic:**
1. **Highest**: Execution stage determines branch should be taken (but wasn't predicted)
2. **Medium**: Execution stage determines branch should not be taken (but was predicted)
3. **Lowest**: Branch predictor's initial prediction from fetch stage

**Optimization**: Branch-related signals are **not** carried beyond Execute stage, eliminating unnecessary pipeline registers in Memory and Writeback stages.

---

## Branch Prediction System

### Branch Predictor Architecture

The branch predictor combines three specialized components to achieve dynamic prediction:

```
┌────────────────────────────────────────────────────────────┐
│                   Branch Predictor                          │
└────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌──────────────┐   ┌──────────────┐
│     BTB       │   │     GHR      │   │     PHT      │
│               │   │              │   │              │
│  3-entry      │   │   3-bit      │   │  8-entry     │
│  Cache        │   │   Shift      │   │  1-bit       │
│               │   │   Register   │   │  Counters    │
└───────────────┘   └──────────────┘   └──────────────┘
        │                   │                   │
        │                   └───────┬───────────┘
        │                           │
        ▼                           ▼
     Branch Target            Prediction Decision
     Address (BTA)            (Taken / Not Taken)
        │                           │
        └─────────────┬─────────────┘
                      ▼
              To PC Multiplexer
```

**Operation Flow:**
1. **Fetch Stage**:
   - PC sent to BTB to check for branch target cache hit
   - Current GHR value indexes into PHT
   - PHT outputs prediction (taken/not taken)
   - If hit AND predicted taken: fetch from BTA
   - If miss OR predicted not taken: fetch from PC+4

2. **Execute Stage**:
   - Actual branch outcome computed
   - Compare with BranchPredicted signal
   - If mismatch: flush pipeline and update predictor
   - BTB updated with new branch targets
   - GHR shifted with actual outcome
   - PHT updated on misprediction

### Branch Target Buffer (BTB)

The BTB implements a **3-entry fully associative cache** for storing branch target addresses.

#### Specifications

| Parameter | Value |
|-----------|-------|
| **Cache Size** | 3 entries |
| **Entry Width** | 40 bits (8-bit tag + 32-bit address) |
| **Organization** | Fully associative |
| **Replacement** | LRU (Least Recently Used) |
| **Update** | On negative clock edge |

#### Port Interface

**Inputs:**
```verilog
input  wire        clk                // Clock signal
input  wire        reset              // Asynchronous reset
input  wire [31:0] pc                 // Current program counter
input  wire [31:0] ALUBranchAddress   // Computed branch target from Execute
input  wire [31:0] pcPlus4            // PC + 4 value
input  wire        branchTakenE       // Actual branch outcome from Execute
input  wire        branchPredictedE   // Prediction made during Fetch
```

**Outputs:**
```verilog
output reg  [31:0] BTA                // Branch target address
output reg         hit                // Cache hit indicator
```

#### Operation Algorithm

```
On negative clock edge:
├─ If RESET:
│  ├─ Clear cache0, cache1, cache2
│  ├─ BTA ← 0
│  └─ hit ← 0
│
├─ Else if (branch was taken BUT not predicted):
│  │  // Update cache with new branch
│  ├─ Find least-recently-used entry
│  ├─ Store {PC[7:0], ALUBranchAddress} in LRU entry
│  ├─ BTA ← 0
│  └─ hit ← 0
│
├─ Else if (PC matches cache0[39:32]):
│  ├─ BTA ← cache0[31:0]
│  ├─ hit ← 1
│  └─ Maintain entry order (cache0 is MRU)
│
├─ Else if (PC matches cache1[39:32]):
│  ├─ BTA ← cache1[31:0]
│  ├─ hit ← 1
│  └─ Swap cache0 ↔ cache1 (promote cache1 to MRU)
│
├─ Else if (PC matches cache2[39:32]):
│  ├─ BTA ← cache2[31:0]
│  ├─ hit ← 1
│  └─ Rotate: cache2→cache0, cache0→cache1, cache1→cache2
│
└─ Else (no match):
   ├─ BTA ← 0
   ├─ hit ← 0
   └─ Preserve all cache entries
```

#### Cache Entry Format

```
┌────────────────────────────────────────┐
│         40-bit Cache Entry             │
├──────────────────┬─────────────────────┤
│  Tag (8 bits)    │  Address (32 bits)  │
│   PC[7:0]        │   Branch Target     │
└──────────────────┴─────────────────────┘
    Bits 39:32           Bits 31:0
```

**Example:**
- Branch at address `0x00001004` jumps to `0x00002050`
- Cache entry: `{0x04, 0x00002050}`

#### LRU Replacement Policy

The BTB uses **position-based LRU**:
- **cache0**: Most recently used (MRU)
- **cache1**: Second most recently used
- **cache2**: Least recently used (LRU)

**On Cache Hit:**
- Hit on cache0: No change
- Hit on cache1: Swap with cache0
- Hit on cache2: Rotate to cache0 position

**On Cache Miss (during branch taken):**
- Overwrite cache2 with new entry
- Demote existing entries

### Global History Register (GHR)

A **3-bit shift register** that maintains a history of recent branch outcomes.

#### Specifications

| Parameter | Value |
|-----------|-------|
| **Width** | 3 bits |
| **Type** | Shift-left register |
| **Update** | Synchronous (on branch execution) |
| **Encoding** | 1 = taken, 0 = not taken |

#### Operation

```
┌─────────────────────────────────────────┐
│     GHR 3-bit Shift Register            │
└─────────────────────────────────────────┘
    MSB                             LSB
     ┌────┬────┬────┐
     │ b2 │ b1 │ b0 │  ← Current state
     └────┴────┴────┘
       │    │    │
       └────┴────┴──────> To PHT (index)

On each branch (when BranchE = 1):
  ┌────┬────┬────┐
  │ b1 │ b0 │ BT │  ← After shift
  └────┴────┴────┘
           └─────── BranchTakenE (new bit)
```

**Example Sequence:**
```
Initial:       GHR = 000 (no branches yet)
Branch 1 (T):  GHR = 001 (taken)
Branch 2 (T):  GHR = 011 (taken, taken)
Branch 3 (NT): GHR = 110 (taken, taken, not taken)
Branch 4 (T):  GHR = 101 (taken, not taken, taken)
```

#### Port Interface

**Inputs:**
```verilog
input wire       clk          // Clock signal
input wire       reset        // Asynchronous reset
input wire       BranchE      // Branch instruction in Execute stage
input wire       BranchTakenE // Actual branch outcome
```

**Outputs:**
```verilog
output reg [2:0] GHR_out      // 3-bit history for PHT indexing
```

#### Control Logic

```
If RESET:
  GHR_out ← 000

Else if (BranchE = 1):  // Branch instruction executing
  GHR_out ← {GHR_out[1:0], BranchTakenE}  // Shift left, insert new bit

Else:
  GHR_out ← GHR_out  // Hold current value
```

### Pattern History Table (PHT)

An **8-entry lookup table** implementing 1-bit saturating counters for branch direction prediction.

#### Specifications

| Parameter | Value |
|-----------|-------|
| **Entries** | 8 (indexed by 3-bit GHR) |
| **Counter Width** | 1 bit per entry |
| **States** | 0 = Not Taken, 1 = Taken |
| **Indexing** | Direct-mapped by GHR value |
| **Update** | Synchronous on misprediction |

#### Structure

```
GHR Value         PHT Entry         Prediction
(Index)           (State)           (Output)
─────────────────────────────────────────────
  000      ─────>  PHT[0]  ─────>    0/1
  001      ─────>  PHT[1]  ─────>    0/1
  010      ─────>  PHT[2]  ─────>    0/1
  011      ─────>  PHT[3]  ─────>    0/1
  100      ─────>  PHT[4]  ─────>    0/1
  101      ─────>  PHT[5]  ─────>    0/1
  110      ─────>  PHT[6]  ─────>    0/1
  111      ─────>  PHT[7]  ─────>    0/1
```

#### Port Interface

**Inputs:**
```verilog
input wire       clk                // Clock signal
input wire       reset              // Asynchronous reset
input wire [2:0] GHR                // Index from Global History Register
input wire       BranchE            // Branch instruction in Execute
input wire       BranchTakenE       // Actual outcome from Execute
input wire       BranchPredictedE   // Prediction from Fetch stage
```

**Outputs:**
```verilog
output wire      prediction         // Asynchronous prediction output
```

#### Prediction Logic (Asynchronous)

```
prediction ← PHT[GHR]  // Direct lookup, no clock delay
```

**This allows fast prediction during Fetch stage!**

#### Update Logic (Synchronous)

```
On positive clock edge:

If RESET:
  ├─ PHT[0] ← 0
  ├─ PHT[1] ← 0
  ├─ ...
  └─ PHT[7] ← 0  // Initialize all to "Not Taken"

Else if (BranchE = 1):  // Branch executing
  │
  ├─ Case 1: Predicted NOT TAKEN, Actually TAKEN
  │  │  Condition: (BranchTakenE = 1) AND (BranchPredictedE = 0)
  │  │  Action: PHT[GHR] ← 1  // Flip to "Taken"
  │
  └─ Case 2: Predicted TAKEN, Actually NOT TAKEN
     │  Condition: (BranchTakenE = 0) AND (BranchPredictedE = 1)
     │  Action: PHT[GHR] ← 0  // Flip to "Not Taken"
```

**Note**: PHT only updates on **mispredictions**. Correct predictions do not change the table.

#### Example Scenario

```
Scenario: Loop with 3 iterations

GHR State    PHT[GHR]    Prediction    Actual    Mispredicted?    Update
─────────────────────────────────────────────────────────────────────────
000          0 (NT)      Not Taken     Taken     YES              PHT[000]←1
001          0 (NT)      Not Taken     Taken     YES              PHT[001]←1
011          0 (NT)      Not Taken     Taken     YES              PHT[011]←1
111          0 (NT)      Not Taken     Not Taken NO               -
110          0 (NT)      Not Taken     -         -                -

After learning, PHT[001] = 1, PHT[011] = 1:
GHR State    PHT[GHR]    Prediction    Actual    Mispredicted?
─────────────────────────────────────────────────────────────
000          1 (T)       Taken         Taken     NO
001          1 (T)       Taken         Taken     NO
011          1 (T)       Taken         Not Taken YES (exit)
```

---

## Hazard Handling

### Data Hazards

Data hazards occur when instructions have register dependencies. The processor employs **forwarding** and **stalling** to resolve these.

#### Forwarding Logic

**Forwarding Sources:**
```
┌─────────────────────────────────────────────────────┐
│              Forwarding Multiplexers                 │
└─────────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  No Forward  │  │Forward from M│  │Forward from W│
│              │  │              │  │              │
│   RD1/RD2    │  │  ALUOutM     │  │  ResultW     │
│  (from Decode│  │  (from Memory│  │ (from Write- │
│   stage)     │  │   stage)     │  │  back stage) │
└──────────────┘  └──────────────┘  └──────────────┘

ForwardA/B = 00   ForwardA/B = 10   ForwardA/B = 01
```

**Forwarding Conditions:**

**Forward A (for RD1 → ALU input A):**
```
If (RA1E = WA3M) AND (RegWriteM = 1) AND (RA1_valid = 1) AND (WA3M_valid = 1):
  ForwardA ← 10  // Forward from Memory stage

Else if (RA1E = WA3W) AND (RegWriteW = 1) AND (RA1_valid = 1) AND (WA3W_valid = 1):
  ForwardA ← 01  // Forward from Writeback stage

Else:
  ForwardA ← 00  // No forwarding
```

**Forward B (for RD2 → ALU input B):**
```
If (RA2E = WA3M) AND (RegWriteM = 1) AND (RA2_valid = 1) AND (WA3M_valid = 1):
  ForwardB ← 10  // Forward from Memory stage

Else if (RA2E = WA3W) AND (RegWriteW = 1) AND (RA2_valid = 1) AND (WA3W_valid = 1):
  ForwardB ← 01  // Forward from Writeback stage

Else:
  ForwardB ← 00  // No forwarding
```

**Validity Checking:**

Validity signals prevent **false hazards**:
- **Invalid register addresses** (e.g., from immediate instructions where RA1/RA2 are not used)
- **Flushed instructions** (validity bits cleared during flush)
- **R0 register** (hardwired to zero, should not participate in forwarding)

```
Validity Signals:
├─ RA1_valid: Set when RA1 contains a valid source register address
├─ RA2_valid: Set when RA2 contains a valid source register address
├─ WA3M_valid: Set when WA3M contains a valid destination register in Memory stage
└─ WA3W_valid: Set when WA3W contains a valid destination register in Writeback stage

On Flush:
  All validity bits in flushed stages ← 0
```

#### Stalling Logic

**Load-Use Hazard:**

When a **load instruction** is followed by a dependent instruction, forwarding alone is insufficient (data not available until Memory stage).

```
Example:
  LDR R1, [R2]      // Cycle N: Load R1 from memory
  ADD R3, R1, R4    // Cycle N+1: Needs R1 value (not ready!)
```

**Stall Condition:**
```
If (MemReadE = 1) AND ((RA1D = WA3E) OR (RA2D = WA3E)):
  StallF ← 1   // Stall Fetch stage
  StallD ← 1   // Stall Decode stage
  FlushE ← 1   // Insert bubble in Execute stage
```

**Stall Effect:**
```
Cycle   F          D          E          M          W
─────────────────────────────────────────────────────
  N     LDR        ...        ...        ...        ...
  N+1   ADD        LDR        ...        ...        ...
  N+2   ADD        LDR        BUBBLE     ...        ...  ← Stall inserted
  N+3   ...        ADD        LDR        BUBBLE     ...
  N+4   ...        ...        ADD        LDR        BUBBLE
```

**Not Implemented:**
- **R15 (PC) write stalls**: Intentionally omitted (would require 4-cycle stall)

### Control Hazards

Control hazards arise from branch and jump instructions that alter the PC.

#### Branch Misprediction Recovery

**Flush Mechanism:**

```
┌───────────────────────────────────────────────────┐
│         Branch Misprediction Detection             │
└───────────────────────────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Case 1:      │  │ Case 2:      │  │ Case 3:      │
│              │  │              │  │              │
│ Predicted NT │  │ Predicted T  │  │ Correct      │
│ Actually T   │  │ Actually NT  │  │ Prediction   │
│              │  │              │  │              │
│ MISPREDICTION│  │ MISPREDICTION│  │ NO ACTION    │
└──────────────┘  └──────────────┘  └──────────────┘
        │               │
        └───────┬───────┘
                ▼
         Flush Pipeline
```

**Flush Conditions:**

**Condition 1: Predicted Not Taken, Actually Taken**
```
If (BranchTakenE = 1) AND (BranchPredictedE = 0):
  FlushD ← 1  // Flush Decode stage
  FlushE ← 1  // Flush Execute stage
  PC ← ALUResultE  // Correct PC to branch target
```

**Condition 2: Predicted Taken, Actually Not Taken**
```
If (BranchTakenE = 0) AND (BranchPredictedE = 1):
  FlushD ← 1  // Flush Decode stage
  FlushE ← 1  // Flush Execute stage
  PC ← PCPlus4E  // Correct PC to sequential address
```

**Flush Implementation:**

Flush signals affect **both datapath registers and control signals**:
```
When FlushD = 1:
  ├─ Instruction in Decode stage ← NOP
  ├─ Control signals in Decode ← 0
  └─ Validity signals in Decode ← 0

When FlushE = 1:
  ├─ Instruction in Execute stage ← NOP
  ├─ Control signals in Execute ← 0
  ├─ Validity signals in Execute ← 0
  └─ BranchPredictedE ← 0
```

#### Register Branch Instructions (BX)

For **BX (Branch and Exchange)** instructions:

```
BX Rm  // Branch to address in Rm

Pipeline Behavior:
  Decode:  Read Rm value
  Execute: Compute branch target (RD2 value)
           FlushD ← 1
           FlushE ← 1
           PC ← RD2
```

**PC+4 Handling:**

PC+4 is carried through pipeline for **BL (Branch with Link)** instructions:
```
BL target  // Branch to target, store return address in R14

Execute Stage:
  R14 ← PC+4  // Return address
  PC ← target
```

---

## Instruction Support

### Data Processing Instructions

**Format:** Register-register or register-immediate operations

#### Register Operations

**Example: ADD**
```
ADD Rd, Rn, Rm  // Rd ← Rn + Rm

Pipeline Flow:
┌──────────┬─────────────────────────────────────────┐
│  Stage   │  Operation                              │
├──────────┼─────────────────────────────────────────┤
│ Decode   │ - RA1 ← Rn, RA2 ← Rm                   │
│          │ - RD1 ← RegFile[Rn]                     │
│          │ - RD2 ← RegFile[Rm]                     │
│          │ - WA3 ← Rd (forwarded to Execute)      │
├──────────┼─────────────────────────────────────────┤
│ Execute  │ - ALU_A ← RD1 (or forwarded value)     │
│          │ - ALU_B ← RD2 (or forwarded value)     │
│          │ - ALUResult ← ALU_A + ALU_B            │
├──────────┼─────────────────────────────────────────┤
│ Memory   │ - ALUOut ← ALUResult                   │
│          │ - No memory access                      │
├──────────┼─────────────────────────────────────────┤
│ Writeback│ - RegFile[Rd] ← ALUOut (falling edge)  │
└──────────┴─────────────────────────────────────────┘
```

#### Immediate Operations (Move Immediate)

**Example: MOV with immediate**
```
MOV Rd, #imm8  // Rd ← imm8 (rotated)

Implementation:
┌──────────────────────────────────────────┐
│ Immediate Encoding (ARM rotate immediate)│
├──────────────────────────────────────────┤
│ - imm8: 8-bit immediate value            │
│ - rot:  4-bit rotation amount            │
│ - Actual value: imm8 ROR (rot × 2)       │
└──────────────────────────────────────────┘

Pipeline Flow:
  Decode:
    - ImmSrc ← 00  // Select imm8
    - ExtImm ← Sign-extend/rotate immediate
    - ALUSrc ← 1   // Select immediate for ALU input B

  Execute:
    - Shifter operation: ROR by (rot << 1)
    - ALU operation: Pass through or OR with 0
    - ALUResult ← rotated immediate value

  Writeback:
    - RegFile[Rd] ← ALUResult
```

**Shifter Configuration:**
```
shamt (shift amount) ← rot << 1  // Multiply rotation by 2
Shift operation ← ROR (Rotate Right)
```

### Memory Operations

#### Load (LDR)

**Format:** `LDR Rd, [Rn, #offset]`

```
Pipeline Flow:
┌──────────┬─────────────────────────────────────────┐
│  Stage   │  Operation                              │
├──────────┼─────────────────────────────────────────┤
│ Decode   │ - RA1 ← Rn (base register)             │
│          │ - RD1 ← RegFile[Rn]                     │
│          │ - ExtImm ← Sign-extend offset           │
│          │ - WA3 ← Rd (destination)               │
├──────────┼─────────────────────────────────────────┤
│ Execute  │ - ALU_A ← RD1                           │
│          │ - ALU_B ← ExtImm                        │
│          │ - ALUResult ← RD1 + ExtImm (address)   │
│          │ - NO shifter operation                  │
├──────────┼─────────────────────────────────────────┤
│ Memory   │ - MemRead ← 1                           │
│          │ - ReadData ← DataMem[ALUResult]        │
│          │ - ALUOut ← ALUResult                   │
├──────────┼─────────────────────────────────────────┤
│ Writeback│ - ResultW ← ReadData                    │
│          │ - RegFile[Rd] ← ResultW                 │
└──────────┴─────────────────────────────────────────┘
```

**Load-Use Hazard:**
```
LDR R1, [R2]    // Load from memory
ADD R3, R1, R4  // Immediately uses R1 → STALL needed
```

#### Store (STR)

**Format:** `STR Rd, [Rn, #offset]`

```
Pipeline Flow:
┌──────────┬─────────────────────────────────────────┐
│  Stage   │  Operation                              │
├──────────┼─────────────────────────────────────────┤
│ Decode   │ - RA1 ← Rn (base register)             │
│          │ - RA2 ← Rd (data to store)             │
│          │ - RD1 ← RegFile[Rn]                     │
│          │ - RD2 ← RegFile[Rd]                     │
│          │ - ExtImm ← Sign-extend offset           │
├──────────┼─────────────────────────────────────────┤
│ Execute  │ - ALU_A ← RD1                           │
│          │ - ALU_B ← ExtImm                        │
│          │ - ALUResult ← RD1 + ExtImm (address)   │
│          │ - Forward RD2 (data value)             │
├──────────┼─────────────────────────────────────────┤
│ Memory   │ - MemWrite ← 1                          │
│          │ - DataMem[ALUResult] ← RD2             │
├──────────┼─────────────────────────────────────────┤
│ Writeback│ - No operation (no register write)     │
└──────────┴─────────────────────────────────────────┘
```

### Branch and Control Flow

#### Branch (B)

**Format:** `B label`

```
Pipeline Flow:
┌──────────┬─────────────────────────────────────────┐
│  Stage   │  Operation                              │
├──────────┼─────────────────────────────────────────┤
│ Fetch    │ - Branch predictor consulted            │
│          │ - If predicted taken: fetch from BTA    │
│          │ - If predicted not taken: fetch PC+4    │
├──────────┼─────────────────────────────────────────┤
│ Decode   │ - RA1 ← R15 (PC register)              │
│          │ - ExtImm ← imm24 × 4 (sign-extended)   │
│          │ - RD1 ← PC value                        │
├──────────┼─────────────────────────────────────────┤
│ Execute  │ - ALUResult ← RD1 + ExtImm             │
│          │ - Evaluate branch condition             │
│          │ - Compare BranchTaken with prediction   │
│          │ - If mismatch: Flush + correct PC       │
├──────────┼─────────────────────────────────────────┤
│ Memory   │ - Memory read (data discarded)         │
├──────────┼─────────────────────────────────────────┤
│ Writeback│ - If branch: PC ← ResultW              │
└──────────┴─────────────────────────────────────────┘
```

**Immediate Encoding:**
- **imm24**: 24-bit signed offset
- **Actual offset**: imm24 × 4 (word-aligned addresses)
- **Target address**: PC + (imm24 × 4)

#### Branch with Link (BL)

**Format:** `BL label`

```
Additional Operation:
  Writeback Stage:
    - RegFile[R14] ← PC+4  // Save return address
    - PC ← target address
```

**PC+4 Forwarding:**
- PC+4 from **Decode stage** is carried through pipeline
- Written to R14 during Writeback stage when `RegWrite=1`

#### Branch and Exchange (BX)

**Format:** `BX Rm`

```
Pipeline Flow:
┌──────────┬─────────────────────────────────────────┐
│  Stage   │  Operation                              │
├──────────┼─────────────────────────────────────────┤
│ Decode   │ - RA2 ← Rm                              │
│          │ - RD2 ← RegFile[Rm]                     │
├──────────┼─────────────────────────────────────────┤
│ Execute  │ - ALUResult ← RD2 (direct transfer)    │
│          │ - FlushD ← 1                            │
│          │ - FlushE ← 1                            │
├──────────┼─────────────────────────────────────────┤
│ Memory   │ - Memory read (data discarded)         │
├──────────┼─────────────────────────────────────────┤
│ Writeback│ - PC ← ALUResult                        │
└──────────┴─────────────────────────────────────────┘
```

**Write Address Selection:**
- **WriteSrc signal**: Selects between R14 and carried WA3 for register write address (A3)
- Carried through pipeline for proper link register handling

---

## Getting Started

### Prerequisites

**Software:**
- **Verilog Simulator**:
  - ModelSim/Questa (recommended for waveform viewing)
  - Icarus Verilog (open-source alternative)
  - Vivado XSim (if using Xilinx tools)
- **Python 3.7+** (for cocotb testbenches)
- **Cocotb**: `pip install cocotb`
- **FPGA Tools** (optional):
  - Xilinx Vivado (for Xilinx FPGAs)
  - Intel Quartus (for Altera/Intel FPGAs)

**Hardware (for FPGA deployment):**
- Any modern FPGA development board
- Tested platforms: Xilinx 7-series and later

### Quick Start

#### 1. Clone Repository

```bash
git clone <repository-url>
cd ARM-Pipelined-Processor-With-Branch-Predictor
```

#### 2. Run Cocotb Tests

**Test BTB:**
```bash
cd tests/btb
make
# View results in terminal
```

**Test GHR:**
```bash
cd tests/ghr
make
# View results in terminal
```

**Test PHT:**
```bash
cd tests/pht
make
```

**Test Complete Branch Predictor:**
```bash
cd tests/branch_predictor
make
```

#### 3. Simulate with ModelSim

```bash
# Compile Verilog files
vlog src/*.v

# Run simulation
vsim -do "run -all" top_module

# View waveforms
vsim -view vsim.wlf
```

#### 4. Synthesize for FPGA

**Using Vivado:**
```tcl
# In Vivado TCL console
read_verilog [glob src/*.v]
synth_design -top top_module -part xc7a35tcpg236-1
opt_design
place_design
route_design
report_utilization
report_timing
write_bitstream top_module.bit
```

---

## Testing and Verification

### Test Strategy

The processor was verified through **multiple complementary approaches**:

1. **Unit Testing**: Individual modules tested with cocotb
2. **Waveform Analysis**: Signal-level verification for timing correctness
3. **Integration Testing**: Complete system tested with real programs
4. **FPGA Validation**: Hardware deployment for final verification

### Cocotb Test Suite

#### Branch Target Buffer (BTB) Test

**Test Coverage:**
- Cache initialization and reset
- Cache hit detection
- Cache miss handling
- LRU replacement policy
- Entry promotion on hit

**Test Results:**

![BTB Test Results](https://github.com/htmos6/Pipelined-Processor/assets/88316097/11a34743-8bfc-4638-92c1-2a141e9abb45)

**Key Test Cases:**
```python
# Test 1: Cache miss
assert btb.hit == 0
assert btb.BTA == 0

# Test 2: First branch learning
btb.branchTakenE = 1
btb.branchPredictedE = 0
# Verify cache updated

# Test 3: Cache hit
btb.pc = previous_branch_pc
assert btb.hit == 1
assert btb.BTA == expected_target
```

#### Global History Register (GHR) Test

**Test Coverage:**
- Shift register operation
- Branch taken/not taken recording
- Reset behavior
- Output to PHT indexing

**Cocotb Test Results:**

![GHR Test Results](https://github.com/htmos6/Pipelined-Processor/assets/88316097/e6742115-3adb-4e21-9608-02911a4d9df9)

**Waveform Analysis:**

![GHR Waveform](https://github.com/htmos6/Pipelined-Processor/assets/88316097/34e7f6cb-297e-405e-939a-8d52f8fab310)

**Key Test Cases:**
```python
# Test sequence: T, T, NT, T
ghr.BranchE = 1
ghr.BranchTakenE = 1  # Taken
# GHR = 001

ghr.BranchTakenE = 1  # Taken
# GHR = 011

ghr.BranchTakenE = 0  # Not Taken
# GHR = 110

ghr.BranchTakenE = 1  # Taken
# GHR = 101
```

#### Pattern History Table (PHT) Test

**Test Coverage:**
- Initialization to "not taken" (0)
- Asynchronous prediction output
- Synchronous update on misprediction
- All 8 PHT entries
- Edge cases (correct predictions should not update)

**Key Test Cases:**
```python
# Test 1: Initial prediction (all 0)
assert pht.prediction == 0

# Test 2: Misprediction (predicted 0, actually 1)
pht.BranchTakenE = 1
pht.BranchPredictedE = 0
# Verify PHT[GHR] flipped to 1

# Test 3: Correct prediction
pht.BranchTakenE = 1
pht.BranchPredictedE = 1
# Verify PHT[GHR] unchanged
```

#### Complete Branch Predictor Test

**Integration Test:**

![Branch Predictor Test](https://github.com/htmos6/Pipelined-Processor/assets/88316097/a86407db-eb00-41ff-8157-4691afe02c6d)

**Test Scenarios:**
1. Cold start (empty BTB, initial predictions)
2. Learning phase (BTB population, PHT training)
3. Steady state (accurate predictions)
4. Misprediction recovery
5. Loop behavior (repeated patterns)

### Datapath Verification

**Waveform Tests:**

![Datapath Test 1](https://github.com/htmos6/Pipelined-Processor/assets/88316097/51dc9126-b642-4706-9112-6ba975a927e1)

![Datapath Test 2](https://github.com/htmos6/Pipelined-Processor/assets/88316097/0261cbe6-b9cb-430e-8b25-4730d52919c5)

**Test Programs:**
- Arithmetic operations with forwarding
- Load-store sequences with hazards
- Branch-heavy code with predictions
- Nested loops
- Function calls (BL/BX)

### Hazard Unit Verification

**Testing Method:** Waveform analysis (cocotb limitations)

**Verified Scenarios:**
1. **RAW Hazard with Forwarding:**
   ```
   ADD R1, R2, R3
   SUB R4, R1, R5  // Forward R1 from Execute→Execute
   ```

2. **Load-Use Hazard with Stall:**
   ```
   LDR R1, [R2]
   ADD R3, R1, R4  // Stall 1 cycle
   ```

3. **Branch Misprediction:**
   ```
   BEQ label       // Predicted taken, actually not taken
   ADD R1, R2, R3  // Flushed instruction
   ```

4. **Multiple Forwarding:**
   ```
   ADD R1, R2, R3
   SUB R4, R1, R1  // Forward R1 to both ALU inputs
   ```

### FPGA Validation

**Platform:** Xilinx FPGA (specific model not specified)

**Validation Results:**
- ✅ **Hazard Unit**: Fully functional with forwarding and stalling
- ✅ **Branch Predictor**: Accurate predictions with proper misprediction recovery
- ✅ **Complete System**: All instructions execute correctly
- ✅ **Performance**: Branch predictor demonstrates measurable CPI improvement

**Test Programs Run on FPGA:**
- Sorting algorithms
- Matrix operations
- Fibonacci sequence
- Prime number generation

---

## Technical Specifications

### Processor Characteristics

| Parameter | Value |
|-----------|-------|
| **ISA** | ARM v7 (subset) |
| **Word Size** | 32 bits |
| **Registers** | 32 general-purpose (R0-R15) |
| **Architecture** | 5-stage pipeline (F/D/E/M/W) |
| **Memory Organization** | Harvard (separate I-mem, D-mem) |
| **Pipeline Type** | In-order, single-issue |
| **Clock Domains** | Single clock, synchronous |
| **Register File Write** | Negative clock edge |

### Branch Predictor Specifications

| Component | Specification |
|-----------|--------------|
| **BTB Entries** | 3 (fully associative) |
| **BTB Entry Size** | 40 bits (8-bit tag + 32-bit address) |
| **BTB Replacement** | LRU (Least Recently Used) |
| **GHR Width** | 3 bits |
| **PHT Entries** | 8 (indexed by GHR) |
| **PHT Counter** | 1-bit saturating counter |
| **Prediction Latency** | 0 cycles (asynchronous PHT read) |

### Hazard Handling Capabilities

| Hazard Type | Resolution Method | Penalty |
|-------------|------------------|---------|
| **RAW (Data)** | Forwarding | 0 cycles |
| **Load-Use** | Stall | 1 cycle |
| **Branch (correct prediction)** | None | 0 cycles |
| **Branch (misprediction)** | Flush + redirect | 2 cycles |

### Performance Metrics

| Metric | Best Case | Worst Case | Notes |
|--------|-----------|------------|-------|
| **CPI** | 1.0 | 3.0 | Depends on branch accuracy |
| **Branch Penalty** | 0 cycles | 2 cycles | With/without misprediction |
| **Load-Use Penalty** | 1 cycle | 1 cycle | Always requires stall |
| **Forwarding Latency** | 0 cycles | 0 cycles | Combinational forwarding |

### FPGA Resource Estimates

**Note**: Actual synthesis not documented. These are **typical estimates** for similar designs.

**Expected Resources (Xilinx 7-series):**

| Resource | Estimated | Notes |
|----------|-----------|-------|
| **LUTs** | 2000-3500 | Including predictor logic |
| **Flip-Flops** | 1000-1500 | Pipeline registers + control |
| **BRAM** | 4-8 blocks | Instruction + data memory |
| **Max Frequency** | 100-200 MHz | Depends on timing closure |

**To get actual data:** Run synthesis and check utilization reports.

---

## Design Decisions

### Why 5-Stage Pipeline?

**Advantages:**
- Balanced pipeline depth (not too shallow, not too deep)
- Clear stage boundaries for educational understanding
- Sufficient for demonstrating all hazard types
- Industry-standard depth for RISC processors

**Trade-offs:**
- More complex than single-cycle (but better performance)
- Simpler than deeper pipelines (easier to understand)

### Why 1-bit PHT Counters?

**Rationale:**
- Simplicity: Easy to understand and implement
- Fast update: Single bit flip on misprediction
- Sufficient for demonstrating branch prediction concepts

**Alternative (not implemented):**
- 2-bit saturating counters (better for loops with occasional exits)

**Trade-off:** 1-bit counters are more susceptible to noise but adequate for educational purposes.

### Why 3-entry BTB?

**Rationale:**
- Small enough to implement efficiently
- Large enough to demonstrate caching and replacement
- Sufficient for small test programs
- Clear LRU replacement visualization

**Production Systems:** Typically use 256-4096 entries

### Why Negative Edge Register Write?

**Reason:** Simplifies timing within single clock cycle

```
Clock Cycle N:
  Rising Edge:  Read registers, Execute, Memory access
  Falling Edge: Write back to register file

Benefits:
  - Avoid read-after-write hazard within same cycle
  - Simplifies control logic
  - Common in educational designs
```

### Why No CSRs or Interrupts?

**Educational Focus:**
- Keep design manageable for learning
- Focus on pipeline and hazards (core concepts)
- CSRs/interrupts are advanced topics

**Extension Path:** Can be added in future iterations

---

## Future Enhancements

### Immediate Improvements

1. **2-bit Saturating Counters in PHT**
   - Better handling of loop exit conditions
   - More robust prediction for complex patterns

2. **Larger BTB**
   - 8-16 entries for better hit rate
   - Set-associative organization

3. **Bimodal Predictor**
   - Separate prediction table indexed by PC
   - Combine with existing correlating predictor

### Advanced Features

4. **Tournament Predictor**
   - Combine multiple predictor types
   - Meta-predictor to select best predictor

5. **Return Address Stack (RAS)**
   - Dedicated stack for function returns
   - Improves prediction for BX LR

6. **CSR Support**
   - System control registers
   - Performance counters

7. **Exception Handling**
   - Interrupt controller
   - Precise exception support

8. **Cache Hierarchy**
   - L1 instruction cache
   - L1 data cache
   - Cache coherency protocol

9. **Superscalar Execution**
   - Dual-issue pipeline
   - Out-of-order execution

10. **Extended ISA**
    - Thumb instruction set
    - NEON SIMD extensions
    - Floating-point support

---

## Contributing

Contributions are welcome! This is an educational project, and improvements are encouraged.

### How to Contribute

1. **Report Issues**: Use GitHub issues for bugs or feature requests
2. **Submit Pull Requests**:
   - Fork the repository
   - Create a feature branch
   - Add tests for new functionality
   - Ensure all tests pass
   - Submit PR with clear description

### Coding Guidelines

- **Verilog Style**: SystemVerilog acceptable, maintain consistency
- **Comments**: Explain "why", not just "what"
- **Modularity**: Keep modules focused and reusable
- **Testing**: Include cocotb tests for new modules
- **Documentation**: Update README for significant changes

### Testing Requirements

Before submitting:
- ✅ All cocotb tests pass
- ✅ Waveform verification for timing-critical paths
- ✅ No synthesis warnings (if targeting FPGA)
- ✅ Documentation updated

---

## License

This project is licensed under the **MIT License**.

```
Copyright (c) 2024 Mehmet Arslan

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

See [LICENSE](LICENSE) file for details.

---

## Author

**Mehmet Arslan**
[@htmos6](https://github.com/htmos6)

Hardware design engineer specializing in:
- Processor architecture and microarchitecture
- Pipelined CPU design and optimization
- Branch prediction and hazard handling
- FPGA implementation and verification
- Digital logic design with Verilog/VHDL
- Hardware verification (cocotb, waveform analysis)

**Technical Skills Demonstrated in This Project:**
- ✅ Advanced pipelining techniques
- ✅ Dynamic branch prediction algorithms
- ✅ Data and control hazard resolution
- ✅ Register forwarding and stalling mechanisms
- ✅ Validity tracking for hazard prevention
- ✅ FPGA validation and testing
- ✅ Cocotb verification methodology
- ✅ Computer architecture principles

**Other Projects:**
- RISC-V RV32I Processor (see profile)
- Digital design tutorials and implementations

---

## Acknowledgments

### Educational Resources
- **Computer Organization and Design** - Patterson & Hennessy
  - Pipeline design principles
  - Hazard handling techniques

- **Computer Architecture: A Quantitative Approach** - Hennessy & Patterson
  - Branch prediction algorithms
  - Performance optimization

### Tools and Frameworks
- **Cocotb**: Python-based verification framework
  - Website: https://www.cocotb.org/
  - Enabled comprehensive unit testing

- **ModelSim/Vivado**: Simulation and synthesis tools
  - Critical for waveform analysis and FPGA deployment

### Community
- **ARM Architecture Reference Manuals**: For instruction set details
- **FPGA Community**: For synthesis and optimization techniques
- **Open-Source Hardware Community**: For inspiration and best practices

---

## References

### ARM Architecture
- [ARM Architecture Reference Manual](https://developer.arm.com/documentation/)
- [ARM Instruction Set Quick Reference](https://developer.arm.com/documentation/qrc0001/m/)
- [ARM Cortex-A Series Programmer's Guide](https://developer.arm.com/documentation/den0013/d/)

### Branch Prediction
- [Two-Level Adaptive Training Branch Prediction (Yeh & Patt, 1991)](https://ieeexplore.ieee.org/document/106976)
- [Branch Target Buffer Design](https://en.wikipedia.org/wiki/Branch_target_predictor)
- [Pattern History Table Fundamentals](https://en.wikipedia.org/wiki/Branch_predictor)

### Pipelining and Hazards
- [Hennessy & Patterson - Appendix C: Pipelining](https://www.elsevier.com/books/computer-architecture/hennessy/978-0-12-383872-8)
- [Data Forwarding Techniques](https://en.wikipedia.org/wiki/Operand_forwarding)
- [Pipeline Hazards Tutorial](https://www.cs.cornell.edu/courses/cs3410/2019sp/schedule/slides/11-pipelinehazards-notes.pdf)

### Verification
- [Cocotb Documentation](https://docs.cocotb.org/)
- [Waveform Debugging Techniques](https://www.design-reuse.com/articles/10488/waveform-based-debug.html)

### Tools
- **Verilog Resources**: https://www.asic-world.com/verilog/
- **Xilinx Vivado**: https://www.xilinx.com/products/design-tools/vivado.html
- **Icarus Verilog** (open-source): http://iverilog.icarus.com/

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| **1.0** | January 2026 | - Initial release<br>- Complete 5-stage pipeline<br>- Branch predictor (BTB+GHR+PHT)<br>- Hazard handling<br>- Cocotb test suite<br>- FPGA validation |

---

**Last Updated**: January 2026
**Project Status**: ✅ Complete and Validated
**Maintenance**: Active

---

*This processor design serves as an educational resource for understanding advanced pipelining techniques, dynamic branch prediction, and hazard handling in modern processor architectures. It demonstrates industry-standard concepts in a clear, implementable form suitable for both learning and as a foundation for more complex processor implementations.*
