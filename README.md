# ARM Pipelined Processor with Branch Predictor

A complete implementation of a 5-stage pipelined ARM processor featuring advanced hazard handling and dynamic branch prediction.

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
  - [Datapath Design](#datapath-design)
  - [Instruction Types](#instruction-types)
  - [Hazard Unit](#hazard-unit)
- [Branch Prediction](#branch-prediction)
  - [Branch Predictor Architecture](#branch-predictor-architecture)
  - [Branch Target Buffer (BTB)](#branch-target-buffer-btb)
  - [Global History Register (GHR)](#global-history-register-ghr)
  - [Pattern History Table (PHT)](#pattern-history-table-pht)
- [Testing and Verification](#testing-and-verification)

## Overview

This project implements a 5-stage pipelined ARM processor based on the classic RISC pipeline design with modifications for enhanced performance. The processor includes sophisticated hazard detection and resolution mechanisms, as well as a dynamic branch predictor to minimize control hazard penalties.

## Features

- **5-Stage Pipeline**: Fetch, Decode, Execute, Memory, Writeback
- **Hazard Handling**:
  - Data forwarding for data hazards
  - Pipeline stalling for load-use hazards
  - Flush mechanism for control hazards
- **Dynamic Branch Prediction**:
  - Branch Target Buffer (BTB) with 3-entry cache
  - Global History Register (GHR) for tracking branch patterns
  - Pattern History Table (PHT) for prediction decisions
- **Comprehensive Instruction Support**:
  - Data processing instructions
  - Memory operations (LDR/STR)
  - Branch instructions (B, BL, BX)
  - Immediate operations

## Architecture

### Datapath Design

The processor datapath includes several key modifications from the base design:

#### Core Components

- **Shifter**: Added before the B port of the ALU module for barrel shift operations
- **Extended Immediate Unit**: Modified to support multiplication by 4 for branch offsets (imm24 × 4)
- **Register File**: Operates on the negative clock edge to simplify timing
- **PC Management**: PC+4 is propagated through the pipeline for branch return addresses
- **Forwarding Paths**: Multiple forwarding paths from Memory and Writeback stages to Execute stage

#### Control Flow

The processor implements a sophisticated PC selection mechanism with priority ordering:

1. **Highest Priority**: ALUResultE (for mispredicted branches that should be taken)
2. **Medium Priority**: Carried PCPlus4 (for mispredicted branches that should not be taken)
3. **Default**: Branch predictor output (BTA if predicted taken, PCPlus4 if predicted not taken)

Branch-related signals are not carried beyond the Execute stage, as they are no longer needed in Memory and Writeback stages.

### Instruction Types

#### Data Processing Instructions

- **Register Selection**: Rd → A1, Rm → A2
- **Pipeline Flow**:
  - **Decode**: RD1, RD2, WA3D, ExtImm stored
  - **Execute**: Shifter and ALU operations performed
  - **Memory**: ALUResult stored (no memory access)
  - **Writeback**: Result written to register file on falling clock edge

#### Move Immediate Instructions

- **Implementation**: Uses immediate bit for data processing path
- **Configuration**:
  - Shift amount (shamt): rot signal shifted left by 1
  - ImmSrc: 00 (selects imm8)
  - ALUSrc: Selects ExtImm
  - Shift operation: ROR (rotate right)

#### Memory Instructions

- **LDR (Load)**:
  - **Decode**: Rn → A1, address calculation prepared
  - **Execute**: Base + Offset calculation
  - **Memory**: Data read from memory
  - **Writeback**: ReadData written to Rd

- **STR (Store)**:
  - **Decode**: Rn → A1, Rd → A2
  - **Execute**: Address calculation (RD1 + ExtImm)
  - **Memory**: RD2 written to calculated address
  - **Writeback**: No operation

#### Branch Instructions

- **B/BL (Branch/Branch with Link)**:
  - **Decode**: R15 selected, imm24 multiplied by 4
  - **Execute**: RD1 + ExtImm calculation
  - **Writeback**: PC updated when PCSrc is high
  - **BL Special**: PC+4 written to R14 (link register)

- **BX (Branch and Exchange)**:
  - **Decode**: Rm → A2, RD2 stored
  - **Execute**: RD2 transferred to ALUResult
  - **Writeback**: ALUResult written to PC

### Hazard Unit

The hazard unit manages both data and control hazards through flush and stall signals applied to both datapath registers and control signals.

#### Data Hazard Handling

**Forwarding Logic**:
- Monitors RA1, RA2, and WA3 signals throughout the pipeline
- Forwards data from Memory or Writeback stages when register dependencies are detected
- Includes validity checking to prevent false matches with invalid register addresses
- Validity signals are flushed during pipeline flushes to avoid matching with R0

**Stalling**:
- Implemented for load-use hazards (LDR followed by dependent instruction)
- R15 write instructions do not trigger 4-cycle stalls

#### Control Hazard Handling

**Register Branch Instructions**:
- PC+4 is carried through the pipeline
- On branch taken in Execute stage: Decode and Execute stages flushed, carried PC+4 selected

**Conditional Branches**:
- Branch predictor determines initial fetch path
- Actual branch outcome verified in Execute stage
- BranchPredicted signal carried through pipeline for misprediction detection

**Flush Mechanism**:

![Flush Mechanism](https://github.com/htmos6/Pipelined-Processor/assets/88316097/4e96a5d5-8b20-4852-a5eb-12824e230bf7)

## Branch Prediction

### Branch Predictor Architecture

The branch predictor combines three components to achieve dynamic branch prediction:

- **BTB (Branch Target Buffer)**: Caches branch target addresses
- **GHR (Global History Register)**: Tracks recent branch outcomes
- **PHT (Pattern History Table)**: Predicts branch direction based on history

The predictor includes a RESET input for system initialization. When mispredictions occur, the Fetch and Decode stages are flushed, and the PC is corrected accordingly.

![Branch Predictor Architecture](https://github.com/htmos6/Pipelined-Processor/assets/88316097/a0f47289-10ef-4975-aaf9-4cf3a54b7a99)

### Branch Target Buffer (BTB)

The BTB implements a 3-entry fully associative cache for branch target addresses.

#### Inputs
- `clk`: Clock signal
- `reset`: Reset signal
- `pc`: Current program counter
- `ALUBranchAddress`: Computed branch target from Execute stage
- `pcPlus4`: PC + 4 value
- `branchTakenE`: Actual branch outcome from Execute stage
- `branchPredictedE`: Prediction made during Fetch stage

#### Outputs
- `BTA`: Branch target address (32-bit)
- `hit`: Cache hit indicator

#### Operation

The BTB operates on the negative clock edge with the following priority:

1. **Reset**: Clear all cache entries, set BTA and hit to default values
2. **Cache Miss + Taken Branch**: Update least-recently-used cache entry with new branch
3. **Cache Hit on Entry 0**: Output target address, maintain entry priority
4. **Cache Hit on Entry 1**: Output target address, promote to most-recently-used (swap with entry 0)
5. **Cache Hit on Entry 2**: Output target address, promote to most-recently-used (swap with entries 0 and 1)
6. **No Match**: Set BTA to 0, hit to 0

Each cache entry stores 40 bits: 8-bit PC tag + 32-bit target address.

![BTB Structure](https://github.com/htmos6/Pipelined-Processor/assets/88316097/c2b2fb00-e8f6-47bc-87bb-c709b0ebea8d)

### Global History Register (GHR)

The GHR is a 3-bit shift register that maintains a history of recent branch outcomes.

#### Operation

- **Shift Left**: On each branch instruction (when BranchE is asserted)
- **Input Bit**: BranchTakenE (1 for taken, 0 for not taken)
- **Output**: 3-bit history value used as PHT index
- **Update**: Synchronous with clock

![GHR Operation](https://github.com/htmos6/Pipelined-Processor/assets/88316097/608b5e86-1fef-4772-ab92-68e71b9dcd2a)

### Pattern History Table (PHT)

The PHT is an 8-entry table (indexed by 3-bit GHR output) implementing 1-bit saturating counters for branch prediction.

#### Structure

- **Size**: 8 entries × 1 bit
- **Index**: 3-bit GHR output
- **Prediction**: 1 = taken, 0 = not taken
- **Initialization**: All entries set to 0 (not taken) on reset

#### Update Logic

The PHT is updated synchronously when mispredictions occur:

**Case 1: Predicted Not Taken, Actually Taken**
- Condition: `branchTakenE = 1` AND `branchPredictedE = 0`
- Action: Flip PHT entry indexed by GHR to 1

**Case 2: Predicted Taken, Actually Not Taken**
- Condition: `branchTakenE = 0` AND `branchPredictedE = 1`
- Action: Flip PHT entry indexed by GHR to 0

![PHT Update Logic](https://github.com/htmos6/Pipelined-Processor/assets/88316097/b93b0a18-7d10-42d9-8463-1bc1c30ea807)

The PHT provides predictions asynchronously based on the current GHR value, allowing fast branch direction prediction during the Fetch stage.

![PHT Structure](https://github.com/htmos6/Pipelined-Processor/assets/88316097/18611511-c59e-499d-92ef-687a521aac45)

## Testing and Verification

### Test Strategy

The processor was verified through multiple testing approaches:

1. **Unit Testing**: Individual modules tested using cocotb
2. **Waveform Analysis**: Signal-level verification for hazard unit and branch predictor
3. **FPGA Deployment**: Full system testing on hardware

### Test Results

#### BTB Cocotb Test
![BTB Test Results](https://github.com/htmos6/Pipelined-Processor/assets/88316097/11a34743-8bfc-4638-92c1-2a141e9abb45)

#### GHR Cocotb Test
![GHR Test Results](https://github.com/htmos6/Pipelined-Processor/assets/88316097/e6742115-3adb-4e21-9608-02911a4d9df9)

#### GHR Waveform Test
![GHR Waveform](https://github.com/htmos6/Pipelined-Processor/assets/88316097/34e7f6cb-297e-405e-939a-8d52f8fab310)

#### Branch Predictor Integration Test
![Branch Predictor Test](https://github.com/htmos6/Pipelined-Processor/assets/88316097/a86407db-eb00-41ff-8157-4691afe02c6d)

#### Datapath Tests
![Datapath Test 1](https://github.com/htmos6/Pipelined-Processor/assets/88316097/51dc9126-b642-4706-9112-6ba975a927e1)
![Datapath Test 2](https://github.com/htmos6/Pipelined-Processor/assets/88316097/0261cbe6-b9cb-430e-8b25-4730d52919c5)

### Verification Status

- **Hazard Unit**: Verified through waveform analysis before and after integration
- **Branch Predictor Modules**: All submodules verified with cocotb tests
- **Complete System**: Successfully tested and validated on FPGA hardware
- **Performance**: Branch predictor demonstrates accurate prediction with proper misprediction recovery

---

*This processor design demonstrates advanced pipelining techniques including data forwarding, hazard detection, and dynamic branch prediction, suitable for educational purposes and as a foundation for more complex processor implementations.*
