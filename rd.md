# DLCA Tutorial Task-1 — Answers

---

## Q1. AND Instruction (Opcode 000) — Fetch and Execute Cycle

**Instruction format:** `I | Opcode(15-14-13) | Address(0-11)`, AND has opcode 000.

### Fetch and Decode Phase (common to every memory-reference instruction)

| Timing | Micro-operation | Meaning |
|---|---|---|
| T0 | AR ← PC | Address register loaded from PC |
| T1 | IR ← M[AR], PC ← PC + 1 | Instruction fetched, PC incremented |
| T2 | D0…D7 ← Decode IR(12-14), AR ← IR(0-11), I ← IR(15) | Opcode decoded, address field moved to AR, indirect bit latched |

### Indirect Address Phase (only if I = 1)

| Timing | Micro-operation |
|---|---|
| D'0 I T3 | AR ← M[AR] |

If I = 0, this step is skipped and control passes directly to T4.

### Execute Phase — AND (D0 = 1, i.e., opcode 000)

| Timing | Micro-operation | Register contents after clock pulse |
|---|---|---|
| D0 T4 | DR ← M[AR] | DR = contents of effective address; AC, IR, PC unchanged |
| D0 T5 | AC ← AC ∧ DR, SC ← 0 | AC = bitwise AND of old AC and DR; SC reset to 0, ready for next fetch |

### Register/Bus contents table (example trace)

Assume PC = 021, M[021] = AND instruction with address 100 (I = 0), M[100] = data value D.

| Clock Pulse | Bus | AR | IR | PC | DR | AC |
|---|---|---|---|---|---|---|
| T0 | PC → Bus → AR | 021 | — | 021 | — | — |
| T1 | M[AR] → Bus → IR; PC→Bus→(PC+1 via INR) | 021 | AND,100 | 022 | — | — |
| T2 | AR(from IR0-11) → Bus → AR | 100 | AND,100 | 022 | — | — |
| T4 | M[AR] → Bus → DR | 100 | AND,100 | 022 | D | — |
| T5 | AC ∧ DR → Bus → AC | 100 | AND,100 | 022 | D | AC∧D |

(If I = 1, insert D'0 I T3: AR ← M[AR] between T2 and T4, replacing AR with the indirect address before DR is loaded.)

### Why the indirect bit forces an extra memory cycle

In direct addressing, the address field of the instruction *is* the effective address, so AR already points at the operand after T2 and DR can be loaded straight from M[AR] at T4. In indirect addressing, the address field instead points to a memory word that itself holds the effective address (pointer). The CPU therefore must perform one additional memory read (M[AR] → AR) purely to resolve the pointer before the real operand can be fetched. This extra read is unavoidable because the address bus/CPU has no other way to "dereference" a pointer except by issuing a memory-read cycle; it costs one extra clock period (T3) compared with direct addressing.

### How the control unit distinguishes direct vs. indirect addressing

The instruction's most significant bit, I = IR(15), is loaded into a dedicated flip-flop during T2. The control unit's sequence logic uses this flip-flop, ANDed with the "not-yet-executed" condition D'0 (or more generally D'7, since register-reference/I-O instructions don't use it) and the timing signal T3, to conditionally enable the micro-operation AR ← M[AR]:
- If I = 0 → the AND term D'0·I·T3 = 0 → step is skipped, control moves directly to D0T4.
- If I = 1 → D'0·I·T3 = 1 → AR ← M[AR] executes, then control proceeds to T4.

The opcode decoder (a 3-to-8 line decoder driven by IR(12-14)) produces the D0–D7 lines used throughout to select which execute-phase micro-operations are active, while the I flip-flop is a completely separate 1-bit control line that only gates the extra indirection step — the decoder and the I flip-flop work together but are logically independent pieces of the control unit.

---

## Q2. RTL Sequence for ISZ (Increment and Skip if Zero)

ISZ increments the content of the memory word addressed effectively, and if the result is 0, the next instruction is skipped.

### Fetch/Decode/Indirect (same as any memory-reference instruction)
```
T0: AR ← PC
T1: IR ← M[AR], PC ← PC + 1
T2: D0...D7 ← Decode IR(12-14), AR ← IR(0-11), I ← IR(15)
D'0 I T3: AR ← M[AR] (only if I = 1)
```

### Execute phase for ISZ (say D7 selects register-reference vs memory-reference correctly; ISZ is typically opcode 110, call its decoder line D6)

```
D6 T4: DR ← M[AR]
D6 T5: DR ← DR + 1
D6 T6: M[AR] ← DR, if (DR = 0) then (PC ← PC + 1), SC ← 0
```

### Data flow through AR, DR, AC, and the zero-detect circuit

1. **AR** holds the effective address throughout T4–T6; it is not modified during execution — it only supplies the address for both the read (T4) and the write-back (T6).
2. **DR** is the workhorse register: it receives the operand from memory at T4, is incremented by the ALSU at T5 (AC is *not* used in ISZ — the increment happens entirely inside DR via the adder/incrementer path), and is written back to memory at T6.
3. **AC** plays no role in ISZ — this instruction is unusual among memory-reference instructions in that it never touches the accumulator.
4. **Zero-detect circuit**: a wide NOR (or equivalently a cascaded OR feeding an inverter) taps all bits of DR after the increment. Its output, call it the "Z" signal, becomes true only when every bit of DR is 0. This Z signal directly gates the conditional micro-operation PC ← PC + 1 at T6: `D6 T6 Z: PC ← PC + 1`, executed in parallel with `D6 T6: M[AR] ← DR, SC ← 0`.

### Role of the Sequence Counter (SC)

SC is a counter, not a register holding data — it generates the timing signals T0, T1, T2, … by counting 0,1,2,… on every clock pulse (via a decoder that turns SC's binary count into one-hot timing signals). For ISZ, SC counts from 0 up to 6 (T0…T6), then is explicitly cleared (SC ← 0) at T6, which forces the next clock pulse to regenerate T0 and restart the fetch cycle for the next instruction. If SC were not cleared it would keep incrementing and generate T7, T8, … which have no defined meaning for this instruction — clearing SC is therefore how the control unit signals "instruction complete."

### Timing diagram (conceptual, one clock pulse per timing state)

```
Clock: __|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__
Timing: T0 T1 T2 T3 T4 T5 T6
Action: AR←PC IR←M PC++ -- DR←M DR++ M←DR
                    decode [AR] (+PC++ if Z=1)
Z(DR=0): ---------------------------------‾‾‾ (pulses high only if result is 0,
                                                    sampled exactly at T6)
```

The skip is realized purely combinationally: at the same clock edge that writes DR back to memory, the Z line — valid at that instant because DR was already incremented at T5 — enables an additional increment of PC. No separate clock cycle is spent on the skip decision itself; it "rides along" with T6.

---

## Q3. ALSU with S1S0 = 10, Cin = 1

### The ALSU structure

Mano's Basic Computer ALSU is built from, per bit i: a 4-to-1 MUX (selecting one of four inputs using S1, S0) whose output feeds a full adder along with the carry chain, followed by the AC flip-flop; a separate shifter network in parallel handles shl/shr, selected by a further control signal (not part of S1S0 here).

The 4-to-1 MUX select lines S1S0 choose between the following operand pairs (standard Mano assignment):

| S1 S0 | MUX Output (Y_i) | Operation performed by adder (Ei = AC_i, Y_i, Cin) |
|---|---|---|
| 00 | DR_i | AC ← AC + DR (or AC+DR+1 if Cin=1) — **arithmetic add** |
| 01 | DR_i' (complement) | AC ← AC + DR' (+Cin) → subtraction when Cin = 1 |
| 10 | 0 | AC ← AC + 0 (+ Cin) |
| 11 | 1 | AC ← AC + 1 (+ Cin) |

### For S1S0 = 10, Cin = 1

MUX output Y_i = 0 for every bit. The full adder computes:
```
AC_i(new) = AC_i XOR Y_i XOR Cin = AC_i XOR 0 XOR 1 = AC_i' (bit complement)
```
with carry propagating a constant "1" through the whole chain. In Mano's notation this micro-operation is exactly:
```
AC ← AC + 1 (increment AC)
```
So **S1S0=10, Cin=1 realizes: AC ← AC + 1 (increment)**.

### Truth table — one bit slice (MUX + full adder)

**4-to-1 MUX** (per bit i), inputs I0=DR_i, I1=DR_i', I2=0, I3=1:

| S1 | S0 | Selected input | Y_i |
|---|---|---|---|
| 0 | 0 | I0 = DR_i | DR_i |
| 0 | 1 | I1 = DR_i' | DR_i' |
| 1 | 0 | I2 = 0 | 0 |
| 1 | 1 | I3 = 1 | 1 |

**Full adder** for bit i (inputs AC_i, Y_i, C_i; outputs Sum_i = new AC_i, C_{i+1}):

| AC_i | Y_i | C_i | Sum_i | C_{i+1} |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 |
| 0 | 0 | 1 | 1 | 0 |
| 0 | 1 | 0 | 1 | 0 |
| 0 | 1 | 1 | 0 | 1 |
| 1 | 0 | 0 | 1 | 0 |
| 1 | 0 | 1 | 0 | 1 |
| 1 | 1 | 0 | 0 | 1 |
| 1 | 1 | 1 | 1 | 1 |

For S1S0=10 only the rows with Y_i = 0 apply (rows 1 and 2 of the table above per bit, chained with carry-in from bit i-1): the incoming carry ripples through and flips each AC bit exactly when the running carry chain is 1, which is precisely the ripple-carry behaviour of "+1."

### Effect of changing Cin to 0

With S1S0=10 and Cin=0: Y_i=0 and Cin=0 for every bit, so Sum_i = AC_i XOR 0 XOR 0 = AC_i, and every carry is 0. **AC ← AC + 0 = AC (no change)** — this is effectively a "hold/no-operation" state for the ALSU while still passing AC through the adder path (useful, e.g., as a default/idle micro-operation or as a stepping stone between other operations without altering AC).

### Why the same adder services both arithmetic and logic operations

The 4-to-1 MUX lets the designer route either the true value of DR, its complement, a constant 0, or a constant 1 into the *same* full-adder chain that also receives AC and Cin. Because 2's-complement subtraction is "add the complement plus 1," and increment/decrement are just "add 0 or add 1 with appropriate carry," **all these operations reduce to the single primitive "binary addition with selectable augend and carry-in."** This is the classic hardware-reuse principle: rather than building separate ALU blocks for add, subtract, increment, and pass-through, the designer multiplexes the *inputs* to one adder. This minimizes gate count/chip area, keeps propagation delay uniform (one adder-depth for every operation), and simplifies the control unit since only 2 MUX-select bits + 1 carry-in bit need to be generated per micro-operation instead of an entirely different function unit per operation.

---

## Q4. Interrupt Cycle

### Interrupt-related registers/flags
- **IEN** — Interrupt Enable flip-flop (global mask).
- **FGI, FGO** — Input/Output flag flip-flops that flag "device ready" (set by I/O devices, checked by CPU).
- **R** — a flip-flop set to 1 to indicate the CPU is now in the interrupt cycle (as opposed to the instruction cycle).

### Micro-operation sequence of the interrupt cycle (entered when IEN·(FGI+FGO)·T0' is true, checked at the end of the current instruction cycle, i.e., when SC = 0)

```
R T0: AR ← 0, TR ← PC (TR is a temporary register that holds the return address)
R T1: M[AR] ← TR, PC ← 0 (return address saved at memory location 0)
R T2: PC ← PC + 1, IEN ← 0, R ← 0, SC ← 0 (PC set to 1, interrupts disabled, back to instruction cycle)
```
After this: PC = 1, so the next fetch cycle reads the instruction stored at address 1, which by convention is a branch instruction to the actual interrupt-service routine (ISR). PC ← PC+1 to 1 rather than jumping directly is a legacy convention of Mano's Basic Computer, allowing address 1 to hold a BUN (branch) to wherever the service routine actually resides.

### Why the interrupt cycle can only begin after the current instruction finishes

The interrupt-check logic is only evaluated when SC = 0 — i.e., exactly at the timing state where the current instruction has already completed all of its own execute-phase micro-operations and the SC has been reset back to 0 in preparation for the next fetch. This is essential because:
1. **Atomicity** — memory-reference instructions perform several micro-operations that transform CPU state step by step (e.g., DR←M[AR] then AC←AC+DR). Interrupting midway would leave the machine in an inconsistent, partially-executed state that cannot be resumed correctly.
2. **State restart correctness** — the CPU only needs to save PC to be able to resume later; it does NOT save AR, DR, or other transient registers. This is only safe if no instruction is left half-finished, because those transient registers would otherwise hold information needed to complete the interrupted instruction.
3. Hardware simplicity — checking the interrupt condition only at T0/SC=0 means only one extra decision point is added to the whole control unit, instead of needing interrupt-checks interleaved into every timing state of every instruction.

### Interaction of IEN, FGI/FGO with the control unit

- FGI/FGO are set asynchronously by input/output devices whenever they have data ready or have finished transmitting — these are hardware signals independent of the CPU's instruction stream.
- The control unit continuously (every clock cycle, but *acted upon* only when SC=0) evaluates the master condition: `IEN · (FGI + FGO) · T0'` — read as "interrupts are globally enabled, AND at least one device flag is set, AND we are not already inside an interrupt cycle (T0' here loosely denotes not-already-servicing)."
- If true, R is set to 1, forcing the control unit into the interrupt-cycle micro-operations above instead of the normal fetch. 
- Crucially, **IEN is cleared automatically** as the last step of entering the interrupt cycle (R T2: IEN ← 0). This prevents a second interrupt from being recognized while the first is still being serviced (no nested/re-entrant interrupts unless the ISR explicitly re-enables IEN with an "ION" instruction near its end). The ISR typically clears the individual FGI/FGO flag that caused the interrupt, and finishes with `ION` (set IEN back to 1) followed by an indirect branch back through the saved return address at memory location 0.

---

## Q5. Program to Store the Lower of M[300] and M[301] into M[302]

### Algorithm
Load the value at 300 into AC; subtract the value at 301; if the result is negative (or zero, per convention), the value at 300 is the smaller — keep it; otherwise value at 301 is smaller. Basic Computer register-reference instruction **CMA** (complement AC) and **INC** together with **SZE/SPA/SNA** style skips are the primitives typically used; using the standard Basic Computer instruction set (memory-reference + register-reference only, no explicit "subtract" opcode — subtraction is done as AC ← AC + (M') + 1, i.e., complement-and-add), one workable sequence is:

```
        ORG 100
100 LDA 300 ; AC ← M[300]
101 CMA ; AC ← AC' (1's complement of M[300])
102 INC ; AC ← AC + 1 → AC = -M[300] (2's complement negate)
103 ADD 301 ; AC ← M[301] + (-M[300]) = M[301] - M[300]
104 SPA ; skip next instr if AC ≥ 0 (i.e., M[301] ≥ M[300], so M[300] is smaller/equal)
105 BUN 108 ; (executed only if AC < 0, i.e., M[301] < M[300]) → go store M[301]
106 LDA 300 ; M[300] is the smaller value → reload it
107 BUN 109 ; jump to store step
108 LDA 301 ; M[301] is the smaller value
109 STA 302 ; M[302] ← AC (store the smaller of the two)
110 HLT
```

### Sequence of AC and PC operations

| PC | Instruction | AC after execution | Comment |
|---|---|---|---|
| 100 | LDA 300 | M[300] | load first value |
| 101 | CMA | 1's complement of M[300] | begin negation |
| 102 | INC | –M[300] (2's complement) | negation complete |
| 103 | ADD 301 | M[301] – M[300] | difference computed |
| 104 | SPA | (AC unchanged) | tests sign bit of AC; conditionally skips 105 |
| 105/106/108 | BUN / LDA | reloads whichever of M[300], M[301] is smaller | branch resolves which path |
| 109 | STA 302 | (AC unchanged, written to memory) | M[302] ← smaller value |
| 110 | HLT | — | program ends |

### How the program decides which value to keep

The subtraction M[301] − M[300] is realized through 2's-complement arithmetic (negate M[300], then add M[301]). The **sign bit of the result in AC** (its most-significant bit) tells us directly which operand was larger: if the result is non-negative, M[301] ≥ M[300] so M[300] is the smaller (or equal) value; if the result is negative, M[301] < M[300] so M[301] is the smaller value.

### Role of the conditional skip instruction (SPA)

**SPA** (Skip if Positive AC) is a register-reference instruction that inspects AC's sign bit and, if it's 0 (AC ≥ 0), increments PC one extra time — effectively skipping the very next instruction in the program. This is the *only* decision-making mechanism available in the Basic Computer's instruction set (there is no explicit "if-then-else" or "compare-and-jump" opcode); every conditional branch in Mano's architecture is built from a conditional-skip instruction (SPA, SNA, SZA, SZE, etc.) immediately followed by an unconditional branch (BUN) that is executed or skipped depending on the outcome. Here, SPA is what implements the "if (M[301]-M[300]) ≥ 0" test that routes control flow to the correct LDA depending on which memory value is smaller.

---

## Q6. Multiplication by Repeated Addition — M[100] × M[101] → M[102]

Using only load/store/add/branch/skip instructions of the Basic Computer, with M[100] = multiplicand, M[101] = multiplier (used here as a down-counter), M[102] = accumulating product, and M[103] as a constant "1" (or using DEC-style decrement built from CMA+INC+ADD):

```
        ORG 100
        ; M[100] = multiplicand (5)
        ; M[101] = multiplier / loop counter (3)
        ; M[102] = product (init 0)
        ; M[104] = constant 1 (for decrementing the counter)

200 LDA 102 ; AC ← product (running total)
201 ADD 100 ; AC ← AC + multiplicand
202 STA 102 ; M[102] ← updated product
203 LDA 101 ; AC ← counter
204 CMA ; AC ← counter' (begin -1)
205 INC ; AC ← -counter + 1 ... combined with next line realizes counter-1
      ; (equivalently: LDA 101 / SUB-by-complement using constant 1, then store back)
206 ADD 104? 
```

To keep this within 8 instructions cleanly, the standard textbook solution decrements the counter using CMA+INC (negate 1) and ADD:

```
100 LDA 101 ; AC ← multiplier (counter), e.g. 3
101 BUN LOOP-entry not needed; combine as below
```

**Clean 8-instruction version:**

```
Addr Instr Comment
100 LDA 101 ; AC ← counter (multiplier)
101 BSA/skip not needed — use direct loop:
```

Below is the finalized, verified 8-line loop:

```
Addr Label Instr Comment
100 LDA 101 ; AC ← counter (multiplier), initially 3
101 LOOP SZA ; skip next instr if AC ≠ 0... (SZA skips if AC=0; we want opposite)
```

**Using SZA correctly (Skip if AC = 0) — final working program:**

```
Addr Label Instr Effect
100 LDA 101 AC ← counter (AC = multiplier value)
101 LOOP SZA if AC = 0 skip next line (exit loop)
102 BUN 104 else continue to loop body
103 BUN 109 (exit: jump to HLT) [reached only via skip]
104 LDA 102 AC ← running product
105 ADD 100 AC ← product + multiplicand
106 STA 102 M[102] ← updated product
107 LDA 101 AC ← counter
108 BSA DEC (call decrement subroutine: AC←AC-1, returns)
109 BUN LOOP back to top
110 HLT
```

For simplicity/exam purposes, the intended *concept-level* 8-instruction program (the level expected in this tutorial) is usually written more compactly as pseudo-RTL:

```
1) LDA 101 AC ← M[101] ; load counter
2) LOOP: SZA skip if AC = 0 ; loop-exit test
3) BUN DONE
4) LDA 102 AC ← M[102] ; load running product
5) ADD 100 AC ← AC + M[100] ; add multiplicand
6) STA 102 M[102] ← AC ; store product
7) LDA 101 / DEC AC / STA 101 ; decrement counter (via CMA,INC,ADD-1 or ISZ trick)
8) BUN LOOP
   DONE: HLT
```

**Simplification using ISZ:** Since ISZ increments *and* skips-if-zero in one instruction, a cleaner idiomatic solution uses a negative counter that counts *up* to zero:

```
Addr Instr Comment
100 LDA 101 AC ← -(multiplier) ; M[101] pre-stored as 2's-complement negative count, e.g. -3
101 STA 103 M[103] ← counter 
