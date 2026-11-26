🇬🇧 English | [🇳🇱 Nederlands](README.nl.md)

# Self-Learning Flow Optimization for OpenQuatt (`η_II,room`)

## Introduction
A heat pump circulates warm water through your radiators or underfloor heating (UFH) to heat your home. The speed of that water flow (in liters per hour) has an impact on electricity consumption:
- **Water flows too slowly:** The water cools down too much along the loop, forcing the heat pump to produce hotter supply water to keep the room warm. Higher supply temperatures increase compressor power consumption.
- **Water flows too fast:** The circulation pump consumes more electrical power and water returns too warm to the heat pump, reducing refrigerant subcooling in the heat exchanger.

Every home has a unique hydraulic layout—radiators, underfloor heating, or a hybrid mix—so a single fixed flow setting is rarely optimal for every installation. This Proof of Concept (PoC) lets Home Assistant **measure and learn** which flow rate achieves the highest whole-system efficiency across complete operating cycles.

---

## 1. Problem Statement & Motivation (Radiators, UFH & Hybrid Mix)
In hydronic heat pump systems, heat output depends on the mean water temperature ($T_{mean} = \frac{T_{supply} + T_{return}}{2}$). Therefore, a larger temperature drop ($\Delta T$) always requires a higher supply temperature ($T_{supply}$) for the same heat output.

The optimal $\Delta T$ window and hydraulic behavior vary by emitter type:

| Emitter Type | Typical Optimal $\Delta T$ | Thermal & Hydraulic Behavior |
| :--- | :--- | :--- |
| **Radiators only** | **5.0 – 6.5 K** | Radiators have relatively low water volume. Too low flow (high $\Delta T$) reduces turbulence in the plate heat exchanger and radiators; too high flow increases cubic pump power and lowers liquid subcooling. |
| **Underfloor Heating (UFH) only** | **3.0 – 5.0 K** | UFH has a large heat-exchange surface and high thermal mass, operating at low supply temperatures (28–35 °C). Higher flow keeps the floor surface uniform and lowers condenser pressure, provided pump power remains low. |
| **Hybrid Mix (Radiators + UFH)** | **4.5 – 6.0 K** *(primary loop)* | Radiators dictate the required supply temperature. A mixing manifold with a secondary pump decouples primary and secondary flows, causing $T_{return}$ to rise rapidly if primary flow is too high. |

Because pipe resistance, pump curves, and emitter surfaces differ per home, a single hardcoded $\Delta T$ target in firmware cannot be optimal for every installation.

---

## 2. Objective Function: Second-Law Efficiency at Room Level ($\eta_{II,room}$)
To fairly compare different flow rates across varying outdoor temperatures and operating conditions, this PoC optimizes for **Second-Law (Exergy) Efficiency between outdoor air and indoor room temperature**:

$$\eta_{II,room} = \frac{\text{COP}_{system}}{\text{COP}_{Carnot,room}} = \frac{Q_{delivered} / E_{total,in}}{\frac{T_{room} + 273.15}{T_{room} - T_{outdoor,ODU}}}$$

### Why $\eta_{II,room}$ instead of raw COP or water-based Carnot?
1. **Weather-normalized:** Dividing by $COP_{Carnot,room}$ removes the primary effect of outdoor temperature fluctuations ($T_{outdoor,ODU}$).
2. **Captures the entire thermal chain:** Unlike Carnot formulas based on $T_{supply}$ (which lower the reference bar when a higher flow reduces $T_{supply}$), $\eta_{II,room}$ evaluates the **end-to-end efficiency** of moving heat from outdoor air into the heated room:
   - Condenser heat transfer (pinch point & subcooling),
   - Radiator / UFH heat transfer (LMTD at a given flow rate),
   - Cubic pump power consumption ($P_{pump}$ included in $E_{total,in}$),
   - Defrost energy losses.

### Numerical Example
At $T_{outdoor,ODU} = 0.0\text{ }^\circ\text{C}$ and $T_{room} = 19.5\text{ }^\circ\text{C}$:
- Theoretical Carnot limit: $COP_{Carnot,room} = \frac{292.65}{19.5} \approx 15.01$
- Measured full-cycle system COP (including pump & defrost): `2.80`
- Second-Law Efficiency: $\eta_{II,room} = \frac{2.80}{15.01} \approx \mathbf{18.7\%}$

---

## 3. Experimental Design & Statistical Validation (Home Assistant PoC)
To avoid transient distortion and false positives from measurement noise, the learning model uses a **cycle-blocked online learning approach**:

1. **Full-Cycle Integration (Defrost-to-Defrost):**
   - Flow rate (`manualIpwm`) is held strictly constant throughout an entire heating cycle.
   - Both delivered thermal energy ($Q_{delivered}$) and Carnot-normalized energy ($E_{Carnot,room}$) are integrated over the complete cycle, including the defrost phase.
   - Cycles shorter than `35 min` or longer than `180 min` are excluded to prevent startup/shutdown bias.
2. **Power Buckets:**
   - Learned metrics are separated into two average thermal load regimes: **Low/Mid ($\le 3.2\text{ kW}$)** and **High ($> 3.2\text{ kW}$)**.
3. **Exploration vs. Exploitation:**
   - Tests discrete flow steps (e.g., `608 iPWM` $\approx 435\text{ L/h}$, `555 iPWM` $\approx 525\text{ L/h}$, `500 iPWM` $\approx 605\text{ L/h}$) by switching only at the start of a defrost cycle.
4. **Online Variance & Significance Testing (Welford's Algorithm):**
   - Tracks cycle count ($n$), running mean ($\mu$), and sum of squared deviations ($M_2$) per bucket and flow step.
   - Standard Error:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - A higher flow step is only selected as the winner once $n \ge 4$ and its mean $\eta_{II,room}$ exceeds the baseline by at least $1.5 \times \sqrt{SE_{baseline}^2 + SE_{candidate}^2}$.

---

## 4. Next Steps for Firmware Consideration
- Collect multi-day baseline and A/B cycle datasets across outdoor temperature regimes and emitter types (radiators, UFH, and hybrid).
- Evaluate whether a lightweight version of cycle-integrated $\eta_{II,room}$ tracking can eventually complement or auto-tune OpenQuatt's flow control parameters (`flowKp`, `flowKi`, or dynamic flow curves).
