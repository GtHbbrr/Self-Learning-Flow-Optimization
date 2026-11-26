🇬🇧 English | [🇳🇱 Nederlands](README.nl.md)

# Self-Learning Flow Optimization for OpenQuatt

## Introduction
A heat pump transfers its heat to the water circulating through your radiators or underfloor heating (UFH). The speed of that water flow (in liters per hour) directly affects supply temperature and electricity consumption:
- **Water flows more slowly:** Putting the same thermal power into less water per hour increases the temperature rise ($\Delta T = \frac{P}{\dot{m} \cdot c}$). This directly pushes up the supply temperature ($T_{supply}$) and condenser pressure—even at minimum compressor speed (30 Hz).
- **Water flows more quickly:** $T_{supply}$ stays lower, but the circulation pump consumes more electricity and water returns warmer to the heat pump, reducing heat-exchanger subcooling.

Every home has a unique hydraulic layout—radiators, underfloor heating, or a hybrid mix—so a single fixed flow setting is rarely optimal for every installation. This Proof of Concept (PoC) lets Home Assistant **measure and learn** which flow rate achieves the highest whole-system efficiency across complete operating cycles.

---

## 1. Problem Statement & Motivation (Radiators, UFH & Hybrid Mix)
In hydronic heat pump systems, heat output depends on the mean water temperature ($T_{mean} = \frac{T_{supply} + T_{return}}{2}$). Therefore, a larger temperature drop ($\Delta T = T_{supply} - T_{return}$) always requires a higher supply temperature ($T_{supply}$) for the same heat output.

The optimal $\Delta T$ window and hydraulic behavior vary by emitter type:

| Emitter Type | Typical Optimal $\Delta T$ | Thermal & Hydraulic Behavior |
| :--- | :--- | :--- |
| **Radiators only** | **5.0 – 6.5 K** | Radiators have relatively low water volume. Too low flow (high $\Delta T$) reduces turbulence in the plate heat exchanger and radiators; too high flow increases cubic pump power and lowers liquid subcooling. |
| **Underfloor Heating (UFH) only** | **3.0 – 5.0 K** | UFH has a large heat-exchange surface and high thermal mass, operating at low supply temperatures (28–35 °C). Higher flow keeps the floor surface uniform and lowers condenser pressure, provided pump power remains low. |
| **Hybrid Mix (Radiators + UFH)** | **4.5 – 6.0 K** *(primary loop)* | Radiators dictate the required supply temperature. A mixing manifold with a secondary pump decouples primary and secondary flows, causing $T_{return}$ to rise rapidly if primary flow is too high. |

Because pipe resistance, pump curves, and emitter surfaces differ per home, a single hardcoded $\Delta T$ target in firmware cannot be optimal for every installation.

---

## 2. Objective Function: Self-Calibrating Room Efficiency (~100% Scale)
To fairly compare different flow rates across varying outdoor temperatures and compressor speeds—without manual calibration constants per emitter type—this PoC uses a **self-calibrating efficiency index**:

$$\text{Reference-COP}_{room}(Hz) = \frac{T_{room} + 273.15}{T_{room} - T_{outdoor,ODU}} \cdot f_{ODU}$$

$$\text{Room Efficiency} = \frac{\text{COP}_{system} / \text{Reference-COP}_{room}(Hz)}{K_{baseline}} \times 100\%$$

Where:
- $T_{room}$: continuously measured indoor room temperature (not a static setpoint, ensuring warm-up and solar gain are accounted for accurately).
- $T_{outdoor,ODU}$: outdoor unit ambient air temperature.
- $f_{ODU}$: calibrated ODU Carnot efficiency factor per compressor frequency (`30–90 Hz`).
- $K_{baseline}$: automatically captured during the first valid baseline cycle, calibrating every home (radiators, UFH, or hybrid) to **100.0%** out of the box.

### Why this metric?
1. **Zero per-home tuning constants:** Works out of the box for radiators, underfloor heating, or hybrid systems by calibrating on the home's own baseline cycle.
2. **Weather- and speed-normalized:** Both outdoor temperature fluctuations ($T_{outdoor,ODU}$) and the natural drop in compressor efficiency at high speeds (80–90 Hz) are normalized.
3. **Captures the entire thermal chain:** Because $T_{supply}$ is excluded from the reference bar, lowering $T_{supply}$ via higher flow is never penalized. Any net gain in heat transfer—minus cubic pump power ($P_{pump}$) and defrost losses—directly appears as a score above `100%`.

---

## 3. Experimental Design & Statistical Validation (Home Assistant PoC)
To avoid transient distortion and false positives from measurement noise, the learning model uses a **cycle-blocked online learning approach**:

1. **Full-Cycle Integration (Defrost-to-Defrost or 90-Minute Window):**
   - Flow rate (`manualIpwm`) is held strictly constant throughout an entire cycle.
   - During freezing conditions, evaluation and switching occur at the start of each defrost cycle. During mild weather (lower compressor speeds of `30–55 Hz` without defrosts), a cycle automatically closes after **90 minutes** of continuous heating.
   - Cycles shorter than `35 min` or longer than `180 min` are excluded to prevent startup/shutdown bias.
2. **Power Buckets:**
   - Learned metrics are separated into two average thermal load regimes: **Low/Mid ($\le 3.2\text{ kW}$)** and **High ($> 3.2\text{ kW}$)**.
3. **Exploration vs. Exploitation:**
   - Tests discrete flow steps (e.g., `608 iPWM` $\approx 435\text{ L/h}$, `555 iPWM` $\approx 525\text{ L/h}$, `500 iPWM` $\approx 605\text{ L/h}$).
4. **Online Variance & Significance Testing (Welford's Algorithm):**
   - Tracks cycle count ($n$), running mean ($\mu$), and sum of squared deviations ($M_2$) per bucket and flow step.
   - Standard Error:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - A higher flow step is only selected as the winner once $n \ge 4$ and its mean Room Efficiency exceeds the baseline by at least $1.5 \times \sqrt{SE_{baseline}^2 + SE_{candidate}^2}$.

---

## 4. Installation (Home Assistant)
1. **Backend (Package):** Place only `packages/openquatt_flow_learning.yaml` in your Home Assistant `/config/packages/` directory (ensure `packages: !include_dir_named packages` is enabled in `configuration.yaml`) and restart Home Assistant.
2. **Dashboard Card:** Open your dashboard in Home Assistant → **Edit** → **+ Add Card** → **Manual** and paste the contents of `dashboard.yaml`. *(Do not place `dashboard.yaml` inside `/config/packages/`).*
3. **Baseline & Start:** Keep `input_boolean.oq_flow_learning_actief` turned off for a few cycles to capture the automatic baseline (`100.0%`), then enable it to start automated cycle testing.
