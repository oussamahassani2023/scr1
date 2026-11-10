# Lab 3 (additional): SCR1 pipeline analysis — command `SLL`

**Student:** Hassani Oussama, group P4119, ITMO University
**Course:** Organization of Computer Systems (SoC), variant 10
**Repo / branch:** `github.com/oussamahassani2023/scr1`, branch `lab3_scr1_pipe_analys`

## 1. Goal
Study how the SCR1 pipeline executes one instruction (`sll`), show it on a waveform, and extend the testbench with a non-synthesizable monitor that detects the instruction and prints debug information (the PC).

## 2. Instruction under study
`sll rd, rs1, rs2` — shift left logical: `rd = rs1 << rs2[4:0]`, zeros shifted in from the right.

| 31..25 | 24..20 | 19..15 | 14..12 | 11..7 | 6..0 |
|---|---|---|---|---|---|
| 0000000 | rs2 | rs1 | 001 | rd | 0110011 |

## 3. Pipeline of SCR1 (what the wave shows)
1. **IFU** — sends `imem_req` + `imem_addr`, receives `imem_resp` + `imem_rdata` from the instruction memory.
2. **IDU** — decodes the word, reads `rs1`, `rs2` from the register file (MPRF).
3. **EXU/IALU** — executes the shift; `curr_pc` is the PC of the instruction in this stage.
4. **Write-back** — result is written to `rd` through the MPRF write port.

## 4. Steps
1. New branch from master: `git checkout -b lab3_scr1_pipe_analys`.
2. Only the `sll` ISA test kept (`rv32ui/sll`), others commented out.
3. Run with waves: `make run_verilator_wf TARGETS="riscv_isa" TRACE=1`.
4. Waveform signals added: `clk`, `curr_pc`, `imem_req`, `imem_addr`, `imem_resp`, `imem_rdata` (+ MPRF read/write signals to see operands and result).
5. Module `scr1_tb_log_cmd` added to the testbench (`src/tb/scr1_top_tb_ahb.sv`).

## 5. Monitor (non-synthesizable)
The monitor stores the address of every accepted instruction request in a small FIFO. When the memory answers, it pops the address, decodes the returned word (`opcode = 0110011`, `funct3 = 001`, `funct7 = 0000000`) and, for `sll`, prints:

```
Detected command: SLL  | PC = 0x........ | instr = 0x........
```

## 6. Results
- Waveform: `report/wave_sll.png`
- Monitor output: `report/log_sll.txt`

## 7. Conclusion
The `sll` instruction was followed through fetch, decode, execute and write-back. The monitor detects it at the instruction-fetch interface and prints its PC. Note that detection happens at fetch, so instructions that are fetched but later flushed (after a jump or trap) can also be reported.
## 8. Waveform observations
- At 5272 ps `imem_resp = 01` and `imem_rdata = 0x002091B3` (`sll gp,ra,sp`): the instruction arrives from instruction memory, and the monitor prints `PC = 0x0000063c`.
- At the same moment `imem_addr` is already `0x00000640` (the next fetch). This is why the monitor stores request addresses in a FIFO instead of using the current `imem_addr`.
- `curr_pc` (execute stage) reaches `0x0000063C` at 5302 ps, 3 clock cycles (30 ps) after the word was fetched.
