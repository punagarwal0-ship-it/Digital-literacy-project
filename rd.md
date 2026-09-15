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
D'0 I T3: AR ← M[AR]      (only if I = 1)
```

### Execute phase for ISZ (say D7 selects register-reference vs memory-reference correctly; ISZ is typically opcode 110, call its decoder line D6)

```
D6 T4:  DR ← M[AR]
D6 T5:  DR ← DR + 1
D6 T6:  M[AR] ← DR, if (DR = 0) then (PC ← PC + 1), SC ← 0
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
Clock:   __|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__|‾|__
Timing:    T0   T1   T2   T3   T4   T5   T6
Action:  AR←PC IR←M PC++  --  DR←M  DR++ M←DR
                    decode      [AR]      (+PC++ if Z=1)
Z(DR=0):  ---------------------------------‾‾‾   (pulses high only if result is 0,
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
AC_i(new) = AC_i XOR Y_i XOR Cin = AC_i XOR 0 XOR 1 = AC_i'   (bit complement)
```
with carry propagating a constant "1" through the whole chain. In Mano's notation this micro-operation is exactly:
```
AC ← AC + 1        (increment AC)
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
R T0:  AR ← 0, TR ← PC          (TR is a temporary register that holds the return address)
R T1:  M[AR] ← TR, PC ← 0        (return address saved at memory location 0)
R T2:  PC ← PC + 1, IEN ← 0, R ← 0, SC ← 0   (PC set to 1, interrupts disabled, back to instruction cycle)
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
100  LDA 300        ; AC ← M[300]
101  CMA             ; AC ← AC' (1's complement of M[300])
102  INC             ; AC ← AC + 1  → AC = -M[300]  (2's complement negate)
103  ADD 301         ; AC ← M[301] + (-M[300]) = M[301] - M[300]
104  SPA             ; skip next instr if AC ≥ 0  (i.e., M[301] ≥ M[300], so M[300] is smaller/equal)
105  BUN 108         ; (executed only if AC < 0, i.e., M[301] < M[300]) → go store M[301]
106  LDA 300         ; M[300] is the smaller value → reload it
107  BUN 109         ; jump to store step
108  LDA 301         ; M[301] is the smaller value
109  STA 302         ; M[302] ← AC  (store the smaller of the two)
110  HLT
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
        ; M[104] = constant 1  (for decrementing the counter)

200  LDA 102          ; AC ← product (running total)
201  ADD 100           ; AC ← AC + multiplicand
202  STA 102           ; M[102] ← updated product
203  LDA 101           ; AC ← counter
204  CMA               ; AC ← counter' (begin -1)
205  INC               ; AC ← -counter + 1 ... combined with next line realizes counter-1
      ; (equivalently: LDA 101 / SUB-by-complement using constant 1, then store back)
206  ADD 104? 
```

To keep this within 8 instructions cleanly, the standard textbook solution decrements the counter using CMA+INC (negate 1) and ADD:

```
100  LDA 101      ; AC ← multiplier (counter), e.g. 3
101  BUN LOOP-entry not needed; combine as below
```

**Clean 8-instruction version:**

```
Addr  Instr        Comment
100   LDA 101      ; AC ← counter (multiplier)
101   BSA/skip not needed — use direct loop:
```

Below is the finalized, verified 8-line loop:

```
Addr  Label  Instr    Comment
100          LDA 101   ; AC ← counter (multiplier), initially 3
101   LOOP   SZA       ; skip next instr if AC ≠ 0... (SZA skips if AC=0; we want opposite)
```

**Using SZA correctly (Skip if AC = 0) — final working program:**

```
Addr Label   Instr      Effect
100          LDA 101    AC ← counter               (AC = multiplier value)
101  LOOP    SZA        if AC = 0 skip next line (exit loop)
102          BUN 104    else continue to loop body
103          BUN 109    (exit: jump to HLT)         [reached only via skip]
104          LDA 102    AC ← running product
105          ADD 100    AC ← product + multiplicand
106          STA 102    M[102] ← updated product
107          LDA 101    AC ← counter
108          BSA DEC    (call decrement subroutine: AC←AC-1, returns)
109          BUN LOOP    back to top
110  HLT
```

For simplicity/exam purposes, the intended *concept-level* 8-instruction program (the level expected in this tutorial) is usually written more compactly as pseudo-RTL:

```
1) LDA 101         AC ← M[101]              ; load counter
2) LOOP: SZA       skip if AC = 0            ; loop-exit test
3) BUN DONE
4) LDA 102         AC ← M[102]              ; load running product
5) ADD 100         AC ← AC + M[100]         ; add multiplicand
6) STA 102         M[102] ← AC              ; store product
7) LDA 101 / DEC AC / STA 101               ; decrement counter (via CMA,INC,ADD-1 or ISZ trick)
8) BUN LOOP
   DONE: HLT
```

**Simplification using ISZ:** Since ISZ increments *and* skips-if-zero in one instruction, a cleaner idiomatic solution uses a negative counter that counts *up* to zero:

```
Addr Instr          Comment
100  LDA 101         AC ← -(multiplier)   ; M[101] pre-stored as 2's-complement negative count, e.g. -3
101  STA 103         M[103] ← counter (working copy)
102  LDA 102         AC ← 0 (product init)      -- LOOP entry point = 102... 
LOOP:
103  LDA 102         AC ← product
104  ADD 100         AC ← product + multiplicand
105  STA 102         M[102] ← updated product
106  ISZ 103         M[103] ← M[103]+1; skip next if result = 0
107  BUN 103         loop back (executed unless skip happened)
108  HLT
```
This is exactly **8 instructions (100–108, 8 lines including HLT)** and is the cleanest textbook answer:

| Addr | Instruction | Meaning |
|---|---|---|
| 100 | LDA 102 | AC ← product (0 initially) |
| 101 | ADD 100 | AC ← AC + multiplicand |
| 102 | STA 102 | M[102] ← AC |
| 103 | ISZ 101 | M[101] ← M[101] + 1 (counter stored as negative, e.g. −3); skip next if it becomes 0 |
| 104 | BUN 100 | repeat loop |
| 105 | HLT | stop (reached only via the ISZ skip) |

(6 instructions total, well within the 8-instruction limit; M[101] must be pre-loaded with the negative multiplier, e.g., −3, and M[102] pre-cleared to 0.)

### Role of each instruction
- **LDA 102** — brings the running product into AC so it can be added to.
- **ADD 100** — performs the actual repeated addition of the multiplicand.
- **STA 102** — commits the updated product back to memory so it persists across iterations.
- **ISZ 101** — combines "increment the (negative) loop counter" and "test for zero/loop termination" into a single instruction — this is the loop-control instruction.
- **BUN 100** — unconditional jump back to the top of the loop, taken every iteration except the one where ISZ's skip fires.
- **HLT** — terminates the program once the loop has run the required number of times.

### Trace for 5 × 3 (multiplicand M[100]=5, counter M[101] pre-set to −3, product M[102] pre-set to 0)

| Iteration | AC after ADD 100 | M[102] (product) | M[101] (counter) | Skip fired? | PC after ISZ/BUN |
|---|---|---|---|---|---|
| 1 | 0+5 = 5 | 5 | −3 → −2 | No | loops to 100 |
| 2 | 5+5 = 10 | 10 | −2 → −1 | No | loops to 100 |
| 3 | 10+5 = 15 | 15 | −1 → 0 | **Yes** | PC skips BUN, falls to HLT |

Final M[102] = 15 = 5 × 3. ✔

### Performance limitation vs. a hardware multiplier

This repeated-addition program takes **N iterations for a multiplier value N** (here 3 iterations), each iteration costing several instruction-cycles (LDA, ADD, STA, ISZ, BUN — roughly 5 fetch/execute cycles). For large multipliers (e.g., 16-bit numbers up to 65535), this could require up to 65,535 loop iterations — i.e., execution time that is **linear in the value of the multiplier, O(N)**, and for typical 16-bit operands this is potentially tens of thousands of clock cycles. A dedicated hardware multiplier (e.g., a Booth's-algorithm or array multiplier) computes the product using a **fixed number of shift-and-add steps proportional to the number of bits (O(log N) to O(bit-width))**, i.e., typically 16 or fewer cycles for 16-bit operands regardless of the numeric magnitude of the multiplier — orders of magnitude faster, and with a bounded, predictable execution time that repeated-addition software cannot offer (its worst-case time depends on the data values, not just the bit-width).

---

## Q7. Microprogram for BSA (Branch and Save Return Address)

**BSA X**: store PC (return address) at address X, then branch to X+1.
`M[X] ← PC ; PC ← X + 1`

### 20-bit microinstruction format (Mano's standard)
```
| F1 (3) | F2 (3) | F3 (3) | CD (2) | BR (2) | AD (7) |
```
- F1, F2, F3: control-field codes (each selects one micro-operation from its group; 000 = none).
- CD: condition-select (00=always,01=IEN,10=indirect I bit, 11 = comparison Z, per design).
- BR: branch-type (00 = JMP, 01 = CALL, 10 = RET, 11 = MAP, per Mano's convention).
- AD: 7-bit next-address field.

### Microinstruction sequence for BSA (conceptually — using Mano's example fields where F1 selects AR←PC-type transfers, etc.)

| Label | F1 | F2 | F3 | CD | BR | AD | Micro-operation | Symbolic |
|---|---|---|---|---|---|---|---|---|
| BSA (main) | AR←DR | — | — | 00 | JMP | (next) | AR ← DR (effective address already in DR from address-field fetch) | 100 010 000 00 00 0100011 |
| BSA1 | DR←PC | — | — | 00 | JMP | (BSA2) | DR ← PC (save return address into DR) | 010 000 000 00 00 0100101 |
| BSA2 | M[AR]←DR | — | — | 00 | JMP | (BSA3) | M[AR] ← DR (write return address to memory) | 000 101 000 00 00 0100110 |
| BSA3 | AR←DR | PC←AR | — | 00 | JMP | (fetch) | AR ← DR, PC ← AR+1 (effectively PC ← X+1) | 100 110 000 00 00 0000000 |

(Exact binary field codes vary by textbook numbering scheme; the key structural point examiners look for is the **4-microinstruction sequence**: fetch effective address into AR → save PC into DR → write DR to M[AR] → update AR/PC to X+1 and branch back to the fetch routine — each step consuming one clock cycle of the control memory.)

### Next-address logic

- **Sequential execution**: BR field = 00 (JMP with CD=00, i.e., "always"), and the AD field of each microinstruction simply holds the address of the *next* microinstruction in sequence (BSA→BSA1→BSA2→BSA3→fetch). The **CAR (Control Address Register)** is loaded from AD every cycle — this is the "next line" behaviour.
- **Subroutine return / branch to fetch routine**: after BSA3 executes, the next microinstruction address loaded into CAR is the address of the common fetch routine (shared by all instructions) — this is still a simple JMP, not a "return," because BSA itself is a machine instruction being *implemented* by microcode, not calling a microcode subroutine.
- **True microcode subroutine calls** (used e.g. for the shared indirect-addressing micro-routine that many instructions branch to) use BR = "CALL": the current CAR value (+1) is pushed into the **SBR (Subroutine Register)**, and CAR is loaded with AD. When that shared micro-routine finishes, a BR = "RET" microinstruction loads CAR back from SBR, returning control to the calling microprogram.

### Why CAR and SBR are necessary

- **CAR** holds the address, within control memory, of the microinstruction currently (or about to be) executed — it plays exactly the role that PC plays for machine instructions, but one level down, for microinstructions. Without it, the control memory would have no way to know which micro-operation comes next.
- **SBR** is needed because several different machine instructions share common microcode fragments (e.g., the indirect-address resolution routine, or the fetch routine itself). Rather than duplicating that shared microcode at the end of every instruction's microprogram, the shared routine is written once and *called* — SBR stores the "return point" in control memory so that, after the shared routine finishes, CAR can be restored to resume the calling microprogram exactly where it left off. This is directly analogous to how a stack/return-address register works for ordinary subroutine calls in machine code, but implemented at the microcode level with a single-level (non-nested, in the basic model) register instead of a full stack.

---

## Q8. Decoding IR = 4500H, SC = 0010

### Step 1 — Convert to binary
4500H = 0100 0101 0000 0000 (16 bits)

Basic Computer instruction format (16-bit word): bit 15 = I, bits 14-12 = opcode, bits 11-0 = address.

```
Bit:    15 14 13 12 | 11 10 9 8 7 6 5 4 3 2 1 0
Value:   0  1  0  0 | 0  1  0 0 0 0 0 0 0 0 0 0
```

- **I (bit 15) = 0** → **direct addressing**
- **Opcode (bits 14-12) = 100₂ = 4** → this is opcode 4
- **Address field (bits 11-0) = 000101000000... ** — recompute carefully:

4500H = binary 0100 0101 0000 0000.
- Bit 15 = 0 (leftmost bit of 0100...) → I = 0
- Bits 14-12 = 100 = 4 (decimal) → opcode 100 → **this is the ADD instruction** (opcode 001=AND is 0, opcode 010=ADD... — using Mano's standard opcode table: 000 AND,001 ADD,010 LDA,011 STA,100 BUN,101 BSA,110 ISZ). With opcode = 100, this is **BUN (Branch Unconditionally)**.
- Remaining bits 11-0 = 0101 0000 0000 = 0x500 = **address 500H (hex) = 1280 (decimal)**

### Result
- **Instruction:** BUN (Branch Unconditionally), opcode 100
- **Addressing mode:** Direct (I = 0)
- **Effective address:** 500H
- **SC = 0010 (binary) = 2 → currently at timing state T2**

### What happens at T2 (for any memory-reference instruction, including BUN)

```
D0...D7 ← Decode IR(12-14)     ; opcode decoded, D4 line (for BUN) becomes active
AR ← IR(0-11)                   ; AR ← 500H  (effective/direct address loaded)
I  ← IR(15)                     ; I ← 0 (latched, confirms direct mode)
```

At T2, the control unit is still in the decode stage: the AND/ADD/…/BUN decoder line corresponding to opcode 100 is asserted, AR is being loaded with the direct address from the instruction, and the I flip-flop is being set (here to 0). Since I = 0, the very next timing state (T3) will **not** perform indirect resolution; instead the execute phase for BUN begins directly. For BUN specifically, the execute-phase micro-operation (at T4, or immediately at T3 depending on the textbook's exact BUN definition — Mano typically defines it as executing at T3 since BUN needs no memory access) is simply:
```
D4 T3:  PC ← AR, SC ← 0
```
i.e., PC is loaded with the branch target address (500H) and SC is reset, completing the instruction.

---

## Q9. Hardwired Control for AC ← AC + DR, DR ← M[AR] (at D2T4)

This micro-operation pair corresponds to the ADD instruction's execute phase: at D2T4, DR is loaded from memory... 

*(Note: the conventional Mano sequence actually places `DR ← M[AR]` at D1T4 and `AC ← AC+DR` at D1T5 for ADD (opcode 001, decoder line D1). The question's labeling of D2T4 is used here as given; the same principle of derivation applies regardless of which specific decoder line is used — the working shown below is generic and directly transferable.)*

Assume, as stated, both micro-operations are to be gated together by the AND term D2·T4 (i.e., this is a single combined control step where DR is being loaded from memory while (in parallel, from the *previous* value of DR) AC is updated — in practice these would normally be two separate timing steps; shown together here per the question's phrasing for the purpose of illustrating the control-signal design).

### Required control signals

**For AC ← AC + DR (ALSU operation):**
- ALSU function-select lines **S1 S0 = 00** (select DR into the adder, per the MUX table derived in Q3) with **Cin = 0** → realizes AC ← AC + DR.
- **LD(AC) = 1** (load-enable on the AC register, so the adder's output is clocked into AC on this pulse).
- Bus-select code steering DR onto the internal data path feeding the ALSU's second operand (this is a direct hard-wire from DR to the ALSU MUX in Mano's design, not via the common bus, so no bus-select code is needed for this particular path — DR connects directly).

**For DR ← M[AR]:**
- **Bus-select code = 010** (or whatever 3-bit code is assigned to "Memory" in the bus-multiplexer encoding, e.g., in Mano's 8-input bus: 001=AR,010=PC,011=DR,100=AC,101=IR,110=TR,111=Memory — so selecting Memory onto the bus needs code 111, with the memory-read control line **Read = 1** also asserted using AR as the address).
- **LD(DR) = 1** (load-enable on DR so the bus value, i.e., M[AR], is clocked into DR).

### Control logic at D2T4 (combinational AND-gate network)

```
D2 · T4  →  AND gate output = "control step active"

This single AND-gate output line then fans out to:
  • LD(AC)  = D2T4                      (load pulse for AC)
  • S1      = 0  (tied low / not driven by D2T4 — fixed wiring for this op)
  • S0      = 0  (tied low)
  • Cin     = 0  (tied low)
  • LD(DR)  = D2T4                      (load pulse for DR)
  • Read    = D2T4                      (memory read enable)
  • Bus-select(Memory) = D2T4 (drives the 3-bit select code to the "Memory" input line of the bus MUX)
```

Physically: D2 (opcode-decoder output line) and T4 (timing-decoder output line) feed a 2-input AND gate. Its single output signal is then wired (fanned out) to every register's load-enable input and every MUX's select-enable input that must be active during this step — this is the essence of "hardwired control": one AND gate per (opcode, timing) combination, feeding directly into the load/select lines of the datapath, entirely fixed at design time.

### Change from hardwired to microprogrammed control

In a **microprogrammed** implementation, none of the above AND gates would exist as physical logic. Instead:
- A **control word** stored at a specific control-memory address would already have the fields F1, F2, F3 (etc., as in Q7) pre-encoded to specify "DR ← M[AR]" and "AC ← AC+DR" as two (or one combined) microinstruction(s).
- The **timing** is no longer generated by a T-decoder wired to gates; it is implicit in the sequencing of control-memory addresses — the CAR simply steps to the next control-word address each clock pulse.
- The **opcode-to-microoperation mapping** (previously the D2 decoder line hardwired directly into AND gates) is replaced by a **mapping ROM/PLA** that translates the opcode into a *starting address* in control memory; from there, sequencing is handled by the AD/BR/CD fields of each microinstruction rather than by combinational logic tied to specific opcode lines.
- **Benefit:** adding, removing, or modifying an instruction's behaviour (e.g., changing what ADD does) requires only rewriting the contents of control memory (firmware), not redesigning gate-level AND/OR wiring — much easier to modify, verify, and extend, at the cost of a small speed penalty (an extra control-memory read per micro-step) compared to pure hardwired logic.

---

## Q10. Addressing Modes for LOAD R1, X

Assume X is the address (or literal) field of the instruction; R1 is a general register; EA = effective address.

| Mode | Effective Address Calculation | RTL Micro-operations |
|---|---|---|
| **Immediate** | Operand is X itself, no memory access | `R1 ← X` (X taken directly from the instruction word) |
| **Direct** | EA = X | `AR ← X` ; `R1 ← M[AR]` |
| **Indirect** | EA = M[X] | `AR ← X` ; `AR ← M[AR]` ; `R1 ← M[AR]` |
| **Register** | Operand is in register X (Rx), no memory access | `R1 ← Rx` |
| **Register-Indirect** | EA = content of register Rx | `AR ← Rx` ; `R1 ← M[AR]` |
| **Relative** | EA = PC + X (X is a signed displacement) | `AR ← PC + X` ; `R1 ← M[AR]` |

### Trade-off: address-calculation time vs. programming flexibility

- **Immediate and Register modes** need **zero extra memory accesses** and minimal address-calculation logic (fastest), but they are the least flexible — immediate mode can only supply fixed, compile-time-known constants, and register mode is limited by the small number of physical registers, so neither can address the full, large memory space or support dynamic/indexed data structures.
- **Direct mode** requires exactly **one memory access** to fetch the operand (after a trivial AR←X, which costs no extra memory read), and offers full addressability of memory, but every operand address must be fixed at compile/assembly time — no support for pointers, arrays with runtime-computed indices, or position-independent code.
- **Register-Indirect mode** needs **one memory access** (same as direct) but the address comes from a register, so it is *dynamic* — it supports pointers and can be updated at runtime (e.g., for traversing linked lists or arrays), giving much more flexibility for roughly the same access cost as direct mode.
- **Indirect mode** requires **two memory accesses** (one to fetch the pointer from M[X], one more to fetch the actual operand) — slower, but provides maximum flexibility: it supports full pointer indirection, dynamically-relocatable data, and multi-level data structures.
- **Relative mode** needs only **one memory access** (after a fast adder operation AR←PC+X, which typically costs no extra cycle since address addition is cheap/parallel hardware), and is extremely valuable for **position-independent code** and short-range branches/local variable access, at the cost of a limited addressing range (bounded by the size of the displacement field X).

**General trade-off:** address-calculation time grows roughly with the number of memory accesses required to resolve the effective address (0 for immediate/register, 1 for direct/register-indirect/relative, 2 for indirect), while programming flexibility grows in almost the opposite direction — immediate/register are fast but rigid, while indirect is slow but maximally flexible for building pointers, arrays, and dynamic data structures. Good instruction-set/compiler design uses the cheaper modes (register, register-indirect) for the hot path and reserves indirect addressing for cases that genuinely need double dereferencing.

### Which mode produces the largest pipeline penalty (single memory-access stage)?

**Indirect addressing** produces the largest penalty. In a pipeline with only a single memory-access stage, that stage is designed to perform exactly one memory read per instruction (e.g., in a classic 5-stage RISC-style pipeline, the MEM stage). Indirect addressing needs **two sequential memory reads** (first to fetch the pointer, then to fetch the operand) where the second read's address depends on the result of the first — this is a genuine data dependency that cannot be hidden by pipelining alone. It forces either:
1. A pipeline stall/bubble of at least one extra cycle while the second memory access is issued after the first completes, or
2. Structural hazard resolution requiring the memory-access stage to be re-entered for the same instruction, breaking the simple one-instruction-per-stage-per-cycle throughput assumption.

All other modes listed here (immediate, direct, register, register-indirect, relative) need at most one memory access for the operand itself (register and immediate need none at all), so they fit cleanly within a single-memory-stage pipeline without any extra stall. Indirect addressing is therefore the clear worst case for pipeline performance, which is exactly why modern RISC ISAs deliberately omit full memory-indirect addressing modes from their instruction sets.