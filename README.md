# CMOS Circuit Design (sky130-style)

This report documents the CMOS design and SPICE simulation activities based on the sky130 workshop flow. These experiments help deepen the understanding of how fundamental transistor-level circuit properties—such as device physics, sizing, and variation—drive the timing behavior that Static Timing Analysis (STA) approximates. By simulating these circuits in SPICE, we can observe the "real" analog behavior (like delays, noise margins, and variation impacts) that STA models, strengthening the intuition behind digital timing concepts.

___
## Experiment 1: MOSFET Behavior & $I_d$ vs $V_{ds}$ Characteristics

### Purpose
To simulate a single NMOS device, sweeping the drain-to-source voltage ($V_{ds}$) for various gate-to-source voltages ($V_{gs}$). This allows for the observation and plotting of the transistor's linear and saturation operating regions.

### SPICE Netlist

```spice
*Model Description
.param temp=27

*Including sky130 library files
.lib "sky130_fd_pr/models/sky130.lib.spice" tt

*Netlist Description
XM1 Vdd n1 0 0 sky130_fd_pr__nfet_01v8 w=5 l=2
R1 n1 in 55
Vdd vdd 0 1.8V
Vin in 0 1.8V

*simulation commands
.op
.dc Vdd 0 1.8 0.1 Vin 0 1.8 0.2

.control

run
display
setplot dc1
.endc

.end
```

### $I_d$ vs $V_{ds}$ Characteristic Curve

![Id vs Vds curves](images/Exp%201%20-%20Id%20vs%20Vds%20curves.svg)

### Analysis

- **Observations:** The plot shows that for a given gate voltage ($V_{gs}$) above the threshold, the drain current ($I_d$) first rises linearly with $V_{ds}$ (Linear Region) and then flattens out, becoming largely independent of $V_{ds}$ (Saturation Region). Higher $V_{gs}$ values result in higher saturation currents.
    
- **Why it happens (Device Physics):** In the linear region, the transistor acts like a voltage-controlled resistor. As $V_{ds}$ increases, the channel at the drain end begins to "pinch off." Once $V_{ds} \ge V_{gs} - V_t$, the channel is pinched off, and the current saturates because it is now limited by the rate at which carriers are injected into the depletion region, a rate controlled by $V_{gs}$.
    
- **How this ties back to STA concepts:** The saturation current ($I_{dsat}$) is the maximum current a transistor can provide to charge or discharge a capacitor. This current directly determines the **drive strength** of a logic gate. STA delay models are fundamentally based on characterizing this drive strength, as it dictates how quickly the gate's output can switch states.

___
## Experiment 2: Threshold Voltage Extraction & Velocity Saturation

### Purpose
To sweep $V_{gs}$ vs. $I_d$ (in the saturation region) to extract the threshold voltage ($V_t$) and observe short-channel effects like velocity saturation and Channel Length Modulation (CLM).

### SPICE Netlist

1. $I_d$ vs $V_{ds}$ Characteristic Curve
	
```spice
    *Model Description
    .param temp=27
    
    *Including sky130 library files
    .lib "sky130_fd_pr/models/sky130.lib.spice" tt
    
    *Netlist Description
    XM1 Vdd n1 0 0 sky130_fd_pr__nfet_01v8 w=0.39 l=0.15
    R1 n1 in 55
    Vdd vdd 0 1.8V
    Vin in 0 1.8V
    
    *simulation commands
    .op
    .dc Vdd 0 1.8 0.1 Vin 0 1.8 0.2
    
    .control
    
    run
    display
    setplot dc1
    .endc
    
    .end
```
    
2. $I_d$ vs $V_{gs}$ Characteristic Curve
    
```spice
    *Model Description
    .param temp=27
    
    *Including sky130 library files
    .lib "sky130_fd_pr/models/sky130.lib.spice" tt
    
    *Netlist Description
    XM1 Vdd n1 0 0 sky130_fd_pr__nfet_01v8 w=0.39 l=0.15
    R1 n1 in 55
    Vdd vdd 0 1.8V
    Vin in 0 1.8V
    
    *simulation commands
    .op
    .dc Vin 0 1.8 0.1
    
    .control
    
    run
    display
    setplot dc1
    .endc
    
    .end
```    

### Plots

1. $I_d$ vs $V_{ds}$ Characteristic Curve
   
    ![Id vs Vds Curve](images/Exp%202%20-%20Id%20vs%20Vds.svg)

2. $I_d$ vs $V_{gs}$ Characteristic Curve

    ![Id vs Vgs Curve](images/Exp%202%20-%20VT.svg)

### Annotations
- Peak Current, $I_{d_{max}}$ = 196.77 μA
- Threshold Voltage, $V_t$ = 0.77 V

### Analysis

- **Observations:** From the $I_d$ vs $V_{gs}$ plot, the transistor turns on sharply after a threshold voltage ($V_t$) of ~0.77V. At high $V_{gs}$, the current increases almost linearly, not quadratically. The $I_d$ vs $V_{ds}$ plot for this short-channel device shows that the curves in saturation have a noticeable positive slope.
    
- **Why it happens (Device Physics):** The linear rise in current at high $V_{gs}$ is due to **velocity saturation**, where high electric fields in the short channel cause carriers to reach their maximum possible speed. The sloped saturation curves are caused by **Channel Length Modulation (CLM)**, where an increasing $V_{ds}$ shortens the effective channel length, thus increasing the current.
    
- **How this ties back to STA concepts:** $V_t$ is a primary parameter in timing models. Process variations cause $V_t$ to shift, which is a major reason for creating different timing corners (slow, typical, fast). Velocity saturation and CLM are complex physical effects that modern STA timing models must accurately account for to match silicon behavior.

___
## Experiment 3: CMOS Inverter - Voltage Transfer Characteristic (VTC)

### Purpose
To build a standard CMOS inverter and generate its VTC to identify the switching threshold ($V_m$).

### SPICE Netlist

```spice
*Model Description
.param temp=27

*Including sky130 library files
.lib "sky130_fd_pr/models/sky130.lib.spice" tt

*Netlist Description
XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=0.84 l=0.15
XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.36 l=0.15
Cload out 0 50fF
Vdd vdd 0 1.8V
Vin in 0 1.8V

*simulation commands
.op
.dc Vin 0 1.8 0.01

.control
run
setplot dc1
display
.endc

.end
```

### Voltage Transfer Characteristic Curve
![Voltage Transfer Characteristic Curve](images/Exp%203%20-%20VTC.svg)

### Annotations
- Switching threshold, $V_m$ = 0.88 V

### Analysis

- **Observations:** The VTC plot shows a sharp, high-gain transition from a high output (1.8V) to a low output (0V). The switching threshold ($V_m$), where $V_{in} = V_{out}$, is 0.88V, which is very close to half the supply voltage ($V_{dd}/2 = 0.9V$), indicating a symmetric inverter.
    
- **Why it happens (Device Physics):** The sharp transition occurs when both NMOS and PMOS transistors are in saturation, providing high voltage gain. The switching threshold $V_m$ is the point where the pull-down current from the NMOS equals the pull-up current from the PMOS. A value near $V_{dd}/2$ indicates that the drive strengths of the two transistors are well-matched.
    
- **How this ties back to STA concepts:** A symmetric VTC leads to balanced rise and fall times and optimal noise margins. For STA, the behavior around the switching threshold is crucial, as the gate's input capacitance (including the Miller capacitance) changes dynamically during the transition, affecting the load seen by the previous stage.

___
## Experiment 4: Transient Behavior - Rise / Fall Delays

### Purpose
To apply a pulse input to the inverter and measure the rise and fall propagation delays ($t_{pr}$ and $t_{pf}$).

### SPICE Netlist

```spice
*Model Description
.param temp=27

*Including sky130 library files
.lib "sky130_fd_pr/models/sky130.lib.spice" tt

*Netlist Description
XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=0.84 l=0.15
XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.36 l=0.15
Cload out 0 50fF
Vdd vdd 0 1.8V
Vin in 0 PULSE(0V 1.8V 0 0.1ns 0.1ns 2ns 4ns)

*simulation commands
.tran 1n 10n

.control
run
.endc

.end
```

### Transient Waveform
![Transient Waveform](images/Exp%204%20-%20Rise%20and%20Fall%20Delay.svg)

### Annotations
- Rise propagation delay, $t_{pr}$ = 0.33 ns
- Fall propagation delay, $t_{pf}$ = 0.29 ns

### Analysis

- **Observations:** The simulation confirms the inverter's logical function and shows finite propagation delays. The fall delay (0.29 ns) is slightly shorter than the rise delay (0.33 ns).
    
- **Why it happens (Device Physics):** The delays are caused by the time it takes the transistor's current to charge ($t_{pr}$) or discharge ($t_{pf}$) the load capacitance. The slight asymmetry is due to the different mobilities of charge carriers: electrons (in NMOS) are faster than holes (in PMOS), making the pull-down action slightly faster than the pull-up action for similarly sized transistors.
    
- **How this ties back to STA concepts:** These propagation delays are the fundamental data points for STA. The process of **cell characterization** involves running thousands of SPICE simulations like this to build the timing libraries (e.g., NLDM lookup tables) that STA tools use to calculate path delays. STA must track rise and fall delays separately to accurately model timing.    

___
## Experiment 5: Noise Margin / Robustness Analysis

### Purpose
To analyze the VTC plot to determine the static noise margins ($NM_L$ and $NM_H$), which quantify the inverter's robustness to noise.

### SPICE Netlist

```spice
*Model Description
.param temp=27

*Including sky130 library files
.lib "sky130_fd_pr/models/sky130.lib.spice" tt

*Netlist Description
XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=1 l=0.15
XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.36 l=0.15
Cload out 0 50fF
Vdd vdd 0 1.8V
Vin in 0 1.8V

*simulation commands
.op
.dc Vin 0 1.8 0.01

.control
run
setplot dc1
display
.endc

.end
```

### Voltage Transfer Characteristic Curve
![Noise Margin](images/Exp%205%20-%20Noise%20margin.svg)

### Annotations
- $NM_L = V_{IL} - V_{OL}$ = 0.78 - 0.12 = 0.66 V
- $NM_H = V_{OH} - V_{IH}$ = 1.7 - 0.98 = 0.72 V

### Analysis

- **Observations:** By analyzing the VTC, the low and high noise margins ($NM_L$ and $NM_H$) are calculated to be 0.66V and 0.72V, respectively. These values are large and relatively symmetric.
    
- **Why it happens (Device Physics):** Noise margins are defined by the "unity gain" points on the VTC ($V_{IL}$ and $V_{IH}$), which enclose the high-gain region. The ideal rail-to-rail output swing (0V to $V_{dd}$) of a CMOS inverter maximizes these margins, making the design robust.
    
- **How this ties back to STA concepts:** While STA focuses on timing, it is complemented by **Signal Integrity (SI) analysis**. The noise margins calculated here serve as the "noise budget." SI tools analyze effects like crosstalk and supply droop and verify that the induced noise on a net does not violate the noise margin of the receiving cell, which would cause a functional failure.

___
## Experiment 6: Power-Supply and Device Variation Studies

### Purpose
To investigate the impact of voltage and process variations on inverter performance, observing the effects on the VTC and propagation delays.

### SPICE Netlist

1. Supply Voltage variation
    
```spice
    *Model Description
    .param temp=27
    
    *Including sky130 library files
    .lib "sky130_fd_pr/models/sky130.lib.spice" tt
    
    *Netlist Description
    XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=1 l=0.15
    XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.36 l=0.15
    Cload out 0 50fF
    Vdd vdd 0 1.8V
    Vin in 0 1.8V
    
    .control
    
    let powersupply = 1.8
    alter Vdd = powersupply
        let voltagesupplyvariation = 0
        dowhile voltagesupplyvariation < 6
        dc Vin 0 1.8 0.01
        let powersupply = powersupply - 0.2
        alter Vdd = powersupply
        let voltagesupplyvariation = voltagesupplyvariation + 1
      end
    
    plot dc1.out vs in dc2.out vs in dc3.out vs in dc4.out vs in dc5.out vs in dc6.out vs in xlabel "input voltage(V)" ylabel "output voltage(V)" title "Inveter dc characteristics as a function of supply voltage"
    
    .endc
    
    .end
```
    
2. Device variation (VTC)
    
```spice
    *Model Description
    .param temp=27
    
    *Including sky130 library files
    .lib "sky130_fd_pr/models/sky130.lib.spice" tt
    
    *Netlist Description
    XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=7 l=0.15
    XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.42 l=0.15
    Cload out 0 50fF
    Vdd vdd 0 1.8V
    Vin in 0 1.8V
    
    *simulation commands
    .op
    .dc Vin 0 1.8 0.01
    
    .control
    run
    setplot dc1
    display
    .endc
    
    .end
```
    
3. Device variation (Transient response)
    
```spice
    *Model Description
    .param temp=27
    
    *Including sky130 library files
    .lib "sky130_fd_pr/models/sky130.lib.spice" tt
    
    *Netlist Description
    XM1 out in vdd vdd sky130_fd_pr__pfet_01v8 w=7 l=0.15
    XM2 out in 0 0 sky130_fd_pr__nfet_01v8 w=0.42 l=0.15
    Cload out 0 50fF
    Vdd vdd 0 1.8V
    Vin in 0 PULSE(0V 1.8V 0 0.1ns 0.1ns 2ns 4ns)
    
    *simulation commands
    .tran 1n 10n
    
    .control
    run
    .endc
    
    .end
```    

### Plots

1. Voltage Transfer Characteristic (Supply Variation)

   ![Voltage Transfer Characteristic (Supply Variation)](images/Exp%206%20-%20Supply%20variation.svg)

2. Voltage Transfer Characteristic (Device Variation)

   ![Voltage Transfer Characteristic (Device Variation)](images/Exp%206%20-%20Device%20variation.svg)

3. Transient Waveform (Device Variation)

   ![Transient Waveform (Device Variation)](images/Exp%206%20-%20Device%20variation%20pulse.svg)

### Annotations
- **Supply Variation:** Shifts in switching threshold    

| $V_{dd}$ (V) | $V_m$ (V) |
| ------------ | --------- |
| 0.8          | 0.458     |
| 1            | 0.531     |
| 1.2          | 0.611     |
| 1.4          | 0.701     |
| 1.6          | 0.79      |
| 1.8          | 0.882     |

- **Device Variation:**
    
    1. Switching threshold, $V_m$ = 0.99 V
	    
    2. Noise Margins
        - $NM_L$ = 0.9 - 0.12 = 0.78 V
        - $NM_H$ = 1.71 - 1.11 = 0.6 V
	    
	3. Propagation Delays
        - Rise delay, $t_{pr}$ = 0.06 ns
        - Fall delay, $t_{pf}$ = 0.29 ns            

### Analysis

- **Observations:** Lowering the supply voltage ($V_{dd}$) reduces the switching threshold and noise margins, and softens the VTC transition. Increasing the PMOS strength relative to the NMOS shifts the switching threshold higher (to ~1.0V), drastically speeds up the rise time ($t_{pr}$), and has little effect on the fall time ($t_{pf}$).
    
- **Why it happens (Device Physics):** A lower $V_{dd}$ reduces the gate overdrive voltage ($V_{gs} - V_t$), which significantly lowers the transistor drive current, slowing down the inverter and reducing its gain. A stronger PMOS requires a higher input voltage to be balanced by the weaker NMOS, shifting $V_m$ up. The stronger PMOS then charges the output much faster, while the NMOS pull-down speed remains unchanged.
    
- **How this ties back to STA concepts:** This is the core reason for **multi-corner STA**. Circuits must be verified at different voltage and process corners. A "slow" corner (low $V_{dd}$, weak transistors) is checked for setup violations, while a "fast" corner (high $V_{dd}$, strong transistors) is checked for hold violations. The device variation demonstrates a "skewed" corner (fast-PMOS/slow-NMOS), which might make a rising-edge path faster but a falling-edge path slower, highlighting the need for detailed, path-specific analysis.

___
## Tabulated Results

- Summary of Nominal Parameters    

| **Parameter**               | **Value** | **Units** |
| --------------------------- | --------- | --------- |
| Extracted $V_t$ (NMOS)      | 0.77      | V         |
| Switching Threshold ($V_m$) | 0.88      | V         |
| Rise Propagation Delay      | 0.33      | ns        |
| Fall Propagation Delay      | 0.29      | ns        |
| $V_{OL}$                    | 0.12      | V         |
| $V_{OH}$                    | 1.7       | V         |
| $V_{IL}$                    | 0.78      | V         |
| $V_{IH}$                    | 0.98      | V         |
| Noise Margin Low ($NM_L$)   | 0.66      | V         |
| Noise Margin High ($NM_H$)  | 0.72      | V         |

- Supply Variation Study Results

| Supply Voltage, $V_{dd}$ (V) | Switching Threshold, $V_m$ (V) |
| ---------------------------- | ------------------------------ |
| 0.8                          | 0.46                           |
| 1                            | 0.53                           |
| 1.2                          | 0.61                           |
| 1.4                          | 0.7                            |
| 1.6                          | 0.79                           |
| 1.8                          | 0.88                           |

- Device Variation Study Results

| **Variation Condition**                | $V_m$**​ (V)** | **Rise Delay ($t_{pr}$​) (ns)** | **Fall Delay ($t_{pf}$​) (ns)** | $NM_L$**​ (V)** | $NM_H$**​ (V)** |
| -------------------------------------- | -------------- | ------------------------------- | ------------------------------- | --------------- | --------------- |
| **Nominal**                            | 0.88           | 0.33                            | 0.29                            | 0.66            | 0.72            |
| **Stronger PMOS (W/L** $\uparrow$**)** | 0.99           | 0.06                            | 0.29                            | 0.78            | 0.6             |

___
## Conclusions

These experiments demonstrate the direct and critical link between fundamental transistor-level physics and high-level timing analysis.

1. The core characteristics of a MOSFET—its threshold voltage, drive current in saturation, and short-channel effects—directly determine the delay and noise margins of a logic gate. SPICE simulation provides the ground truth for this behavior.
    
2. Static Timing Analysis is not arbitrary; it is a sophisticated abstraction built upon models that are pre-characterized using SPICE. The propagation delays and transition times measured in these experiments are precisely the data that populate the timing libraries STA relies on.
    
3. The final experiment highlights that a circuit's performance is not a single number. It varies significantly with operating voltage and manufacturing inconsistencies. This is why multi-corner STA is essential for modern chip design. Analyzing performance at nominal, slow, and fast corners ensures that a design will be reliable and meet its timing goals across the full spectrum of real-world conditions.    

___
## References 

- _sky130 Circuit Design Workshop_. GitHub. Retrieved from https://github.com/kunalg123/sky130CircuitDesignWorkshop/
