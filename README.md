# Synchronous FIFO — Verilog Design & Verification

A parameterized synchronous First-In First-Out (FIFO) buffer implemented in SystemVerilog, complete with a self-checking testbench and simulation artifacts from Synopsys VCS.

---

## Overview

A FIFO (First-In First-Out) buffer is a fundamental digital design component used to transfer data between two parts of a system, typically to handle differences in data-production and data-consumption rates. This implementation is **synchronous** — all read and write operations are controlled by a single shared clock edge, making it suitable for single-clock-domain designs.

---
---

## Design — `synchronous_fifo` (`design.sv`)

### Module Interface

| Port        | Direction | Width         | Description                          |
|-------------|-----------|---------------|--------------------------------------|
| `clk`       | input     | 1-bit         | Clock signal                         |
| `rst_n`     | input     | 1-bit         | Active-low synchronous reset         |
| `w_en`      | input     | 1-bit         | Write enable                         |
| `r_en`      | input     | 1-bit         | Read enable                          |
| `data_in`   | input     | `DATA_WIDTH`  | Data to write into the FIFO          |
| `data_out`  | output    | `DATA_WIDTH`  | Data read from the FIFO              |
| `full`      | output    | 1-bit         | Asserted when the FIFO is full       |
| `empty`     | output    | 1-bit         | Asserted when the FIFO is empty      |

### Parameters

| Parameter    | Default | Description                          |
|--------------|---------|--------------------------------------|
| `DEPTH`      | 8       | Number of entries in the FIFO        |
| `DATA_WIDTH` | 8       | Bit-width of each data entry         |

### How It Works

The FIFO uses a circular buffer of `DEPTH` entries with two pointers:

- **`w_ptr`** — write pointer, advances on every successful write
- **`r_ptr`** — read pointer, advances on every successful read

**Write:** On a rising clock edge, if `w_en` is high and the FIFO is not `full`, `data_in` is stored at `fifo[w_ptr]` and `w_ptr` is incremented.

**Read:** On a rising clock edge, if `r_en` is high and the FIFO is not `empty`, `fifo[r_ptr]` is placed on `data_out` and `r_ptr` is incremented.

**Status flags** are computed combinationally:
```
full  = (w_ptr + 1 == r_ptr)   // Next write would catch the read pointer
empty = (w_ptr == r_ptr)        // Pointers are equal — nothing to read
```

**Reset:** An active-low reset (`rst_n = 0`) synchronously clears both pointers and `data_out` to zero.

### Pointer Width

The pointer width is automatically computed as `$clog2(DEPTH)` bits, so it scales correctly with any power-of-two depth.

---

## Testbench — `sync_fifo_TB` (`testbench.sv`)

The testbench instantiates `synchronous_fifo` and runs two concurrent `initial` blocks:

- **Write thread:** Enables `w_en` on even iterations, writes random 8-bit data using `$urandom`, and pushes each written value into a reference queue (`wdata_q`).
- **Read thread:** Enables `r_en` on even iterations, pops the expected value from the reference queue, and compares it against `data_out` using `$error` / `$display`.

The check is:
```
if (data_out !== wdata)
    $error("Comparison Failed: expected %h, got %h", wdata, data_out);
else
    $display("Comparison Passed: wr_data = %h, rd_data = %h", wdata, data_out);
```

A VCD waveform file (`dump.vcd`) is generated automatically via `$dumpfile` / `$dumpvars` for post-simulation analysis.

**Clock period:** 10 ns (toggled every 5 ns).

---

## Simulation

### Tool

- **Simulator:** Synopsys VCS X-2025.06-SP1 (64-bit)
- **Timescale:** `1ns / 1ns`
- **UVM library:** IEEE UVM (bundled with VCS)

### Running the Simulation

```bash
bash run.sh
```

This script:
1. Sets up the environment (VCS paths, UVM home, license server).
2. Compiles `design.sv` and `testbench.sv` with SystemVerilog support (`-sverilog`).
3. Runs the compiled simulation binary (`./simv`).
4. Packages results into `result.zip`.

### Expected Output

```
Time = 215: Comparison Passed: wr_data = xx and rd_data = xx
Time = 225: Comparison Passed: wr_data = xx and rd_data = xx
...
$finish called
```

All comparisons should pass. Any `$error` message indicates a functional mismatch.

### Viewing Waveforms

The simulation produces `dump.vcd`. Open it with any VCD viewer:

```bash
# GTKWave (open-source)
gtkwave dump.vcd
```

Signals available in the waveform:
- `clk`, `rst_n`, `w_en`, `r_en`
- `data_in[7:0]`, `data_out[7:0]`
- `full`, `empty`
- Internal: `w_ptr[2:0]`, `r_ptr[2:0]`, `fifo[0:7]`

---

## Design Notes & Limitations

- **Single clock domain only.** For cross-clock-domain use, a Gray-code pointer based asynchronous FIFO is needed.
- **No overflow/underflow protection in hardware.** The `full` and `empty` flags are provided for the controller to check; the design does not internally guard against writing when full or reading when empty.
- **Depth must be a power of two** for the pointer wraparound (via natural overflow of the `$clog2`-sized register) to work correctly.
- **Registered output.** `data_out` is registered (updated on the clock edge), so there is one cycle of read latency.

---

## Customization

To change the FIFO depth or data width, override the parameters at instantiation:

```verilog
synchronous_fifo #(
    .DEPTH(16),
    .DATA_WIDTH(32)
) my_fifo (
    .clk(clk), .rst_n(rst_n),
    .w_en(w_en), .r_en(r_en),
    .data_in(data_in), .data_out(data_out),
    .full(full), .empty(empty)
);
```
