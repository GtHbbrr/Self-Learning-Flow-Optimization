# Self-Learning Flow Optimization for OpenQuatt (`η_II,room`)

## 1. Problem Statement & Motivation
In hydronic heat pump systems (especially radiator-based setups), selecting the optimal water flow rate is a non-trivial trade-off:
- **Too low flow (`~435 L/h`):** Increases radiator $\Delta T$ (7–9 K), forcing a higher supply temperature ($T_{\text{supply}}$) and higher condenser pressure ($P_{\text{cond}}$), which reduces compressor isentropic and volumetric efficiency.
- **Too high flow (`>700 L/h`):** Increases cubic circulation pump power ($P_{\text{pump}}$), raises return temperature ($T_{\text{return}}$), reduces liquid refrigerant subcooling in the plate heat exchanger, and can cause flow noise.

Static curves or pure $\Delta T$-based PID controllers do not account for installation-specific radiator surface area, hydraulic resistance, or actual whole-system energy efficiency across defrost cycles.

---

## 2. Objective Function: Second-Law Efficiency at Room Level ($\eta_{\text{II,room}}$)
To fairly compare different flow rates across varying outdoor temperatures and operating conditions, this PoC optimizes for **Second-Law (Exergy) Efficiency between outdoor air and indoor room temperature**:

$$\eta_{\text{II,room}} = \frac{\text{COP}_{\text{system}}}{\text{COP}_{\text{Carnot,room}}} = \frac{Q_{\text{delivered}} / E_{\text{total,in}}}{\frac{T_{\text{room}} + 273.15}{T_{\text{room}} - T_{\text{outdoor,ODU}}}}$$

### Why $\eta_{\text{II,room}}$ instead of raw COP or water-based Carnot?
1. **Weather-normalized:** Dividing by $\text{COP}_{\text{Carnot,room}}$ removes the primary effect of outdoor temperature fluctuations ($T_{\text{outdoor,ODU}}$).
2. **Captures the entire thermal chain:** Unlike Carnot formulas based on $T_{\text{supply}}$ (which lower the reference bar when a higher flow reduces $T_{\text{supply}}$), $\eta_{\text{II,room}}$ evaluates the **end-to-end efficiency** of moving heat from outdoor air into the heated room:
   - Condenser heat transfer (pinch point & subcooling),
   - Radiator heat transfer (LMTD at a given flow rate),
   - Cubic pump power consumption ($P_{\text{pump}}$ included in $E_{\text{total,in}}$),
   - Defrost energy losses.

### Numerical Example
At $T_{\text{outdoor,ODU}} = 0.0\text{ }^\circ\text{C}$ and $T_{\text{room}} = 19.5\text{ }^\circ\text{C}$:
- Theoretical Carnot limit: $\text{COP}_{\text{Carnot,room}} = \frac{292.65}{19.5} \approx 15.01$
- Measured full-cycle system COP (including pump & defrost): `2.80`
- Second-Law Efficiency: $\eta_{\text{II,room}} = \frac{2.80}{15.01} \approx \mathbf{18.7\%}$

---

## 3. Experimental Design & Statistical Validation (Home Assistant PoC)
To avoid transient distortion and false positives from measurement noise, the learning model uses a **cycle-blocked online learning approach**:

1. **Full-Cycle Integration (Defrost-to-Defrost):**
   - Flow rate (`manualIpwm`) is held strictly constant throughout an entire heating cycle.
   - Both delivered thermal energy ($Q_{\text{delivered}}$) and Carnot-normalized energy ($E_{\text{Carnot,room}}$) are integrated over the complete cycle, including the defrost phase.
   - Cycles shorter than `35 min` or longer than `180 min` are excluded to prevent startup/shutdown bias.
2. **Power Buckets:**
   - Learned metrics are separated into two average thermal load regimes: **Low/Mid ($\le 3.2\text{ kW}$)** and **High ($> 3.2\text{ kW}$)**.
3. **Exploration vs. Exploitation:**
   - Tests discrete flow steps (e.g., `608 iPWM` $\approx 435\text{ L/h}$, `555 iPWM` $\approx 525\text{ L/h}$, `500 iPWM` $\approx 605\text{ L/h}$) by switching only at the start of a defrost cycle.
4. **Online Variance & Significance Testing (Welford's Algorithm):**
   - Tracks cycle count ($n$), running mean ($\mu$), and sum of squared deviations ($M_2$) per bucket and flow step.
   - Standard Error:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - A higher flow step is only selected as the winner once $n \ge 4$ and its mean $\eta_{\text{II,room}}$ exceeds the baseline by at least $1.5 \times \sqrt{SE_{\text{baseline}}^2 + SE_{\text{candidate}}^2}$.

---

## 4. Next Steps for Firmware Consideration
- Collect multi-day baseline and A/B cycle datasets across outdoor temperature regimes.
- Evaluate whether a lightweight version of cycle-integrated $\eta_{\text{II,room}}$ tracking can eventually complement or auto-tune OpenQuatt's flow control parameters (`flowKp`, `flowKi`, or dynamic flow curves).
