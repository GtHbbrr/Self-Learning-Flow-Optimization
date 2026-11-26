#### Self-Learning Flow Optimization for OpenQuatt (η_II,room)

#### 1. Problem Statement & Motivation

In hydronic heat pump systems (especially radiator-based setups), selecting the optimal water flow rate is a non-trivial trade-off: 

* **Too low flow (~435 L/h):** Increases radiator Δ T (7–9 K), forcing a higher supply temperature (

Tsupplycap T sub supply end-sub
𝑇supply
⁠) and higher condenser pressure (

Pcondcap P sub cond end-sub
𝑃cond
⁠), which reduces compressor isentropic and volumetric efficiency.
* **Too high flow (>700 L/h):** Increases cubic circulation pump power (

Ppumpcap P sub pump end-sub
𝑃pump
⁠), raises return temperature (

Treturncap T sub return end-sub
𝑇return
⁠), reduces liquid refrigerant subcooling in the plate heat exchanger, and can cause flow noise.

Static curves or pure Δ T-based PID controllers do not account for installation-specific radiator surface area, hydraulic resistance, or actual whole-system energy efficiency across defrost cycles. 

#### 2. Objective Function: Second-Law Efficiency at Room Level (

ηII,roometa sub II,room end-sub
𝜂II,room
⁠)

To fairly compare different flow rates across varying outdoor temperatures and operating conditions, this PoC optimizes for **Second-Law (Exergy) Efficiency between outdoor air and indoor room temperature**: 

ηII,room=COPsystemCOPCarnot,room=Qdelivered/Etotal,inTroom+273.15Troom−Toutdoor,ODUeta sub II,room end-sub equals the fraction with numerator COP sub system end-sub and denominator COP sub Carnot,room end-sub end-fraction equals the fraction with numerator cap Q sub delivered end-sub / cap E sub total,in end-sub and denominator the fraction with numerator cap T sub room end-sub plus 273.15 and denominator cap T sub room end-sub minus cap T sub outdoor,ODU end-sub end-fraction end-fraction
𝜂II,room=COPsystemCOPCarnot,room=𝑄delivered/𝐸total,in𝑇room+273.15𝑇room−𝑇outdoor,ODU
 

#### Why 

ηII,roometa sub II,room end-sub
𝜂II,room
⁠ instead of raw COP or water-based Carnot?

1. **Weather-normalized:** Dividing by 

COPCarnot,roomCOP sub Carnot,room end-sub
COPCarnot,room
⁠ removes the primary effect of outdoor temperature fluctuations (

Toutdoor,ODUcap T sub outdoor,ODU end-sub
𝑇outdoor,ODU
⁠).
2. **Captures the entire thermal chain:** Unlike Carnot formulas based on 

Tsupplycap T sub supply end-sub
𝑇supply
⁠ (which lower the reference bar when a higher flow reduces 

Tsupplycap T sub supply end-sub
𝑇supply
⁠), 

ηII,roometa sub II,room end-sub
𝜂II,room
⁠ evaluates the **end-to-end efficiency** of moving heat from outdoor air into the heated room: 

  * Condenser heat transfer (pinch point & subcooling),
  * Radiator heat transfer (LMTD at a given flow rate),
  * Cubic pump power consumption (

Ppumpcap P sub pump end-sub
𝑃pump
⁠ included in 

Etotal,incap E sub total,in end-sub
𝐸total,in
⁠),
  * Defrost energy losses.

#### Numerical Example

At 
𝑇outdoor,ODU

=0.0

 ∘C
⁠ and 
𝑇room

=19.5

 ∘C
⁠: 

* Theoretical Carnot limit: 
COPCarnot,room

=292.6519.5

≈15.01
⁠
* Measured full-cycle system COP (including pump & defrost): 2.80
* Second-Law Efficiency: 
𝜂II,room

=2.8015.01

≈𝟏𝟖.𝟕%
⁠

#### 3. Experimental Design & Statistical Validation (Home Assistant PoC)

To avoid transient distortion and false positives from measurement noise, the learning model uses a **cycle-blocked online learning approach**: 

1. **Full-Cycle Integration (Defrost-to-Defrost):** 

  * Flow rate (manualIpwm) is held strictly constant throughout an entire heating cycle.
  * Both delivered thermal energy (

Qdeliveredcap Q sub delivered end-sub
𝑄delivered
⁠) and Carnot-normalized energy (

ECarnot,roomcap E sub Carnot,room end-sub
𝐸Carnot,room
⁠) are integrated over the complete cycle, including the defrost phase.
  * Cycles shorter than 35 min or longer than 180 min are excluded to prevent startup/shutdown bias.
2. **Power Buckets:** 

  * Learned metrics are separated into two average thermal load regimes: **Low/Mid (≤ 3.2 kW)** and **High (> 3.2 kW)**.
3. **Exploration vs. Exploitation:** 

  * Tests discrete flow steps (e.g., 608 iPWM ≈ 435 L/h, 555 iPWM ≈ 525 L/h, 500 iPWM ≈ 605 L/h) by switching only at the start of a defrost cycle.
4. **Online Variance & Significance Testing (Welford's Algorithm):** 

  * Tracks cycle count (n), running mean (μ), and sum of squared deviations (M₂) per bucket and flow step.
  * Standard Error: 
𝑆𝐸

=𝑀2𝑛(𝑛−1)
⁠
  * A higher flow step is only selected as the winner once n ≥ 4 and its mean 

ηII,roometa sub II,room end-sub
𝜂II,room
⁠ exceeds the baseline by at least **
1.5

×𝑆𝐸2baseline+𝑆𝐸2candidate
⁠**.

#### 4. Next Steps for Firmware Consideration

* Collect multi-day baseline and A/B cycle datasets across outdoor temperature regimes.
* Evaluate whether a lightweight version of cycle-integrated 

ηII,roometa sub II,room end-sub
𝜂II,room
⁠ tracking can eventually complement or auto-tune OpenQuatt's flow control parameters (flowKp, flowKi, or dynamic flow curves).
