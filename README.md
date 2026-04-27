# FPGA-Integrated-Environmental-Monitoring-system
Environmental monitoring
This module is a **Serial Peripheral Interface (SPI) Master** designed to transmit 8 bits of data to a peripheral (like the OLED display in your previous code) using **SPI Mode 0**.

It acts as a translator, taking a parallel 8-bit byte and "clocking it out" one bit at a time over a single wire.

---

##  The Timing Logic 
The module is designed to run on a **100 MHz** system clock but communicate at **1 MHz**. To achieve this, it uses a clock divider.

The logic follows this formula:

$$\text{Toggle Interval} = \frac{f_{system}}{2 \times f_{spi}} = \frac{100\text{ MHz}}{2 \times 1\text{ MHz}} = 50\text{ cycles}$$

Because the `sclk` (Serial Clock) must go high then low to complete one cycle, the code toggles the signal every **50 clock cycles** (`CLK_DIV = 50`).

---

##  State Machine Breakdown
The module uses a **Finite State Machine (FSM)** to manage the transmission flow.

### 1. `S_IDLE`
* **Waiting:** The module sits here until the `start` signal pulses high.
* **Initialization:** When triggered, it loads the `data_in` into an internal `shift` register and pulls `cs` (Chip Select) **low** to tell the peripheral "get ready."

### 2. `S_LOAD`
* **Setup:** This is a brief buffer state. It puts the very first bit (the Most Significant Bit, or MSB) onto the `mosi` (Master Out Slave In) line so it's stable before the clock starts ticking.

### 3. `S_SEND` (The Workhorse)
This state handles the actual toggling of the clock and shifting of bits:
* **Clock Toggling:** Every time `clk_cnt` hits 49, it flips the state of `sclk`.
* **SPI Mode 0 Behavior:**
  * **Rising Edge:** The peripheral samples (reads) the bit on the `mosi` line.
  * **Falling Edge:** The master (this module) shifts the next bit from the register onto the `mosi` line.
* **Counter:** It repeats this until all 8 bits are sent (`bit_cnt` reaches 1).

### 4. `S_DONE`
* **Cleanup:** It pulls `cs` back **high** (de-asserting the peripheral), resets `sclk` to low, and pulses the `done` signal for one cycle to tell the rest of the FPGA, "I'm finished; send me the next byte!"

---

##  SPI Mode 0 Specifics
In **Mode 0** (CPOL=0, CPHA=0), the timing is very specific:
1. The clock stays **low** when idle.
2. Data must be "valid" (stable) on the wire **before** the rising edge of the clock.
3. The slave device captures the data exactly when the clock transitions from **low to high**.

---

##  Key Signals Summary

| Signal | Direction | Purpose |
| :--- | :--- | :--- |
| `data_in[7:0]` | Input | The byte you want to send (e.g., a command for the OLED). |
| `start` | Input | The "Go" button. |
| `done` | Output | The "Finished" notification. |
| `sclk` | Output | The 1 MHz heartbeat for the peripheral. |
| `mosi` | Output | The data stream (1 bit at a time). |
| `cs` | Output | The "Attention" signal for the peripheral (Active Low). |

> **Note:** This is a **Transmit-Only** master. It does not have a `miso` (Master In Slave Out) pin, meaning it can "talk" to sensors or screens, but it cannot "listen" to them.

This module is a custom clock generator that uses the Xilinx-specific **MMCM (Mixed-Mode Clock Manager)** primitive to synthesize a precise **148.5 MHz** clock from a **100 MHz** source. This is the standard clock frequency required for 1080p @ 60Hz video timing.

Instead of using the Vivado GUI "Clocking Wizard," this code manually instantiates the `MMCME2_BASE` hardware block found inside the Artix-7 or Zynq-7000 FPGA fabric.

---

##  The Clock Synthesis Math
The MMCM works by using a Voltage Controlled Oscillator (VCO). It divides the input, multiplies it up to a high frequency, and then divides it back down to the target.

The output frequency ($F_{out}$) is calculated using the following formula:

$$F_{out} = \left( \frac{F_{in}}{DIVCLK\_DIVIDE} \right) \times \frac{CLKFBOUT\_MULT\_F}{CLKOUT0\_DIVIDE\_F}$$

Plugging in the values from your code:
* **Input Division:** $100\text{ MHz} / 5 = 20\text{ MHz}$ (This is the reference frequency for the Phase-Frequency Detector).
* **Multiplication (VCO):** $20\text{ MHz} \times 37.125 = 742.5\text{ MHz}$ (The VCO must stay within a specific range, usually 600 MHz–1200 MHz).
* **Output Division:** $742.5\text{ MHz} / 5 = 148.5\text{ MHz}$.

---

##  Component Breakdown

### 1. `MMCME2_BASE`
This is the hard-IP block inside the FPGA.
* **`CLKIN1_PERIOD`:** Tells the FPGA that the incoming clock is 10ns (100 MHz).
* **`BANDWIDTH`:** Set to "OPTIMIZED" to allow the MMCM to balance jitter and phase noise automatically.
* **`LOCKED`:** This output is crucial. It stays low while the clock is unstable and turns high only when the frequency is locked and safe to use for logic.

### 2. Feedback Loop (`bufg_fb`)
The MMCM requires a feedback path (`CLKFBIN`) to compare the output phase with the input phase.
* By passing the feedback through a **BUFG** (Global Clock Buffer), the MMCM compensates for the internal delay of the clock distribution network, ensuring the output is phase-aligned.

### 3. Output Buffer (`bufg_out`)
The raw output of the MMCM cannot drive thousands of flip-flops directly.
* The BUFG takes the `clk_out_unbuffered` signal and puts it onto a specialized Global Clock Tree in the FPGA. This ensures the clock reaches every part of the chip at the exact same time (minimizing clock skew).
