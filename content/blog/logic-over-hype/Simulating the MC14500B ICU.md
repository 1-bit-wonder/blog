---
title: "Logic Over Hype: Simulating the MC14500B ICU in Rust"
date: "2025-12-27"
category: "Systems Engineering"
tags: ["Rust", "CPU-Design", "Simulation", "Retro-Computing"]
---

Sometimes, to really understand how things work, you have to go back to the metal. After spending significant time in the higher levels of the software stack, I felt an itch to understand the fundamental "contract" between software and silicon. My journey into the depths started with **Nand2Tetris**, where I built a computer from the ground up using HDL. But I wanted to take those first principles and apply them to a real-world piece of history: the **Motorola MC14500B**.

### The Industrial Ghost in the Machine

The MC14500B wasn't designed to run an OS or render graphics. It was an **Industrial Control Unit (ICU)**—the "brain" for systems that didn't need a full 8-bit CPU but required something smarter than a wall of physical relays. In the late 70s and 80s, you would find this 1-bit wonder deep inside:

* **Programmable Logic Controllers (PLCs):** Managing factory automation where bit manipulation was the primary task.
* **Traffic Light Controllers:** Handling the simple on/off timing sequences and sensor inputs at intersections.
* **HVAC Systems:** Managing thermostats and logic in industrial buildings.
* **Assembly Line Logic:** Controlling "pick and place" machinery where rugged, deterministic reliability was paramount.

### A Note on Hardware Parity: The External PC

One of the most fascinating architectural constraints of this chip is that it doesn't actually have an internal **Program Counter (PC)** or an address bus. In a real-world rig, the MC14500B was effectively a "brain without legs." It relied on an external circuit—typically a chain of discrete CMOS counters—to step through instructions. 

In my simulator, I’ve implemented the PC internally within the `Cpu` struct to create a runnable system, but I’ve kept the execution logic decoupled to respect that original "brain-only" architecture.

## The Heartbeat: The `step` Function

The core of the simulator is the `step` method. Unlike a standard interpreter, this method represents a single cycle of the machine, handling the interaction between the Logic Unit (LU) and the CPU's control flags.

## Technical Deep Dive: Enforcing Hardware Constraints

Below are the key implementations where software meets silicon logic.

### 1. Enforcing the Gates: `LU::execute`
The Logic Unit is responsible for all bitwise operations. However, it is strictly governed by the **IEN (Input Enable)** flag. If the gate is closed, the **Result Register (RR)** is immutable.

```rust
pub fn execute(&mut self, code: Opcode, data: bool) {
    let logical_result = match code {
        Opcode::LD => data,
        Opcode::LDC => !data,
        Opcode::AND => self.result_reg & data,
        Opcode::ANDC => self.result_reg & !data,
        Opcode::OR => self.result_reg | data,
        Opcode::ORC => self.result_reg | !data,
        Opcode::XNOR => !(self.result_reg ^ data),
        _ => self.result_reg,
    };

    // Hardware Gating: Only update RR if Input Enable is active.
    if self.ien {
        self.result_reg = logical_result;
    }
}
```

### 2. Output Inhibition: The STO Instruction

While logic operations happen in the LU, data only exits the CPU via the STO (Store) instruction. Here, the OEN (Output Enable) flag acts as a hardware-level inhibitor. If oen is false, the write signal never reaches the output bus.

```Rust
Opcode::STO => {
    if self.lu.oen {
        output = Some(self.lu.result_reg);
    }
}
```

### 3. Branch-less Control: The SKZ Logic

The MC14500B lacks traditional jump instructions. Instead, it uses SKZ (Skip next instruction if Zero). In my step function, I implemented this by manually incrementing the Program Counter if the Result Register is low, effectively "jumping" over the next opcode in the vector.

```Rust
Opcode::SKZ => {
    if !self.lu.result_reg {
        self.pc += 1; // Skip the next instruction cycle
    }
}
```

## Observability via Trace Mode
In main.rs, I implemented a comprehensive Trace Mode using the console and dialoguer crates. It provides a cycle-by-cycle look into the machine's state, printing the PC, current Opcode, Data Bus state, and the internal registers (result_reg, ien, oen) with color-coded boolean values. It transforms a silent loop into an observable system.

```Plaintext
Cycle 0  | PC: 0  | Opcode: LDC  | Data: false | RR: true  | IEN: true | OEN: true | 
Cycle 1  | PC: 1  | Opcode: STO  | Data: false | RR: true  | IEN: true | OEN: true | -> OUTPUT: true
```

## Why Rust?
Rust was a non-negotiable choice for this journey. Its ownership model provides the memory safety required for systems engineering without the "hiccups" of a garbage-collected language. It allowed me to build a deterministic system that respects the physical constraints of the hardware it simulates.

**What's Next:** I’m currently architecting a Distributed CAN BUS Simulation in Rust to explore multi-node communication and resource arbitration. If you’re into low-level tech or protocol design, stay tuned.

Explore the Repo: [github.com/1-bit-wonder/cpu-1bw14500b](https://github.com/1-bit-wonder/cpu-1bw14500b)
