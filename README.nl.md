[🇬🇧 English](README.md) | 🇳🇱 Nederlands

# Zelflerende Flow-optimalisatie voor OpenQuatt (`η_II,kamer`)

## Inleiding
Een warmtepomp pompt warm water door je radiatoren of vloerverwarming om je huis te verwarmen. Hoe snel dat water stroomt (de **flow**, in liter per uur), heeft invloed op je stroomverbruik:
- **Stroomt het water te langzaam?** Dan koelt het water onderweg te veel af. De warmtepomp moet dan heter water maken om de kamer toch warm te krijgen. Heter water kost de compressor extra stroom.
- **Stroomt het water te snel?** Dan verbruikt de waterpomp zelf meer stroom en komt het water te warm terug bij de warmtepomp. Daardoor kan de warmtepomp zijn warmte minder goed kwijt.

Elk huis is anders. Het ene huis heeft alleen radiatoren, het andere vloerverwarming, en weer een ander huis heeft allebei. Daarom werkt één vaste instelling niet voor iedereen het beste. 

Dit project laat Home Assistant **zelf meten en leren** welke watersnelheid in jouw huis het meeste rendement oplevert. Het systeem test verschillende standen steeds over een volledige werkcyclus en kiest pas een nieuwe stand als de metingen zeker weten dat die zuiniger is.

---

## 1. Probleemstelling & Motivatie (Radiatoren, VVW en Mix)
Bij watergedragen warmtepompsystemen is het kiezen van de optimale waterflow een afweging tussen warmteoverdracht en pompenergie. Omdat de warmteafgifte wordt bepaald door de gemiddelde watertemperatuur ($T_{gem} = \frac{T_{aanvoer} + T_{retour}}{2}$), dwingt een hogere $\Delta T$ altijd een hogere aanvoertemperatuur ($T_{aanvoer}$) af.

Het optimale $\Delta T$-venster en hydraulische gedrag verschuiven per afgiftesysteem:

| Systeemtype | Typische optimale $\Delta T$ | Thermisch & hydraulisch gedrag |
| :--- | :--- | :--- |
| **Alleen radiatoren** | **5,0 – 6,5 K** | Radiatoren hebben relatief weinig waterinhoud. Bij te lage flow (hoge $\Delta T$) neemt de turbulentie in de platenwisselaar en radiatoren af; bij te hoge flow stijgt het pompverbruik en daalt de *subcooling*. |
| **Alleen vloerverwarming (VVW)** | **3,0 – 5,0 K** | VVW heeft een groot afgifteoppervlak en grote thermische massa, waardoor $T_{aanvoer}$ al laag ligt (28–35 °C). Een hogere flow houdt de vloer gelijkmatiger warm en verlaagt de condensordruk, zolang het kubische pompverbruik beperkt blijft. |
| **Mix (radiatoren + VVW)** | **4,5 – 6,0 K** *(primair circuit)* | De radiatoren bepalen hier de benodigde aanvoertemperatuur. Een mengverdeler met eigen pomp trekt de primaire en secundaire flow uit elkaar, waardoor $T_{retour}$ snel oploopt als het primaire debiet te hoog wordt. |

Omdat leidingweerstand, pompcurve en afgifte-oppervlak per woning verschillen, is één vast $\Delta T$-getal in firmware nooit voor elke woning ideaal.

---

## 2. Doelfunctie: Second-Law Efficiency op kamerniveau ($\eta_{II,kamer}$)
Om verschillende flows eerlijk te vergelijken bij wisselende buitentemperaturen en bedrijfsomstandigheden, optimaliseert deze Proof of Concept (PoC) op de **Second-Law (exergie) efficiëntie tussen de buitenlucht en de kamertemperatuur**:

$$\eta_{II,kamer} = \frac{\text{COP}_{systeem}}{\text{COP}_{Carnot,kamer}} = \frac{Q_{geleverd} / E_{totaal,in}}{\frac{T_{kamer} + 273{,}15}{T_{kamer} - T_{buiten,ODU}}}$$

### Waarom $\eta_{II,kamer}$ in plaats van ruwe COP of Carnot op aanvoertemperatuur?
1. **Weersgenormaliseerd:** Delen door $COP_{Carnot,kamer}$ haalt het primaire effect van schommelingen in de buitentemperatuur ($T_{buiten,ODU}$) uit de vergelijking.
2. **Omvat de volledige thermische keten:** In tegenstelling tot Carnot-formules op basis van $T_{aanvoer}$ (die de referentielat verlagen zodra een hogere flow $T_{aanvoer}$ doet dalen), beoordeelt $\eta_{II,kamer}$ het **totaalrendement** van warmtetransport van buitenlucht naar de verwarmde kamer:
   - Warmteoverdracht in de condensor (*pinch point* & *subcooling*),
   - Warmteoverdracht in radiatoren en/of VVW (LMTD bij de gekozen flow),
   - Kubisch pompvermogen ($P_{pomp}$ inbegrepen in $E_{totaal,in}$),
   - Energieverlies tijdens ontdooicycli (*defrosts*).

### Rekenvoorbeeld
Bij $T_{buiten,ODU} = 0{,}0\text{ }^\circ\text{C}$ en $T_{kamer} = 19{,}5\text{ }^\circ\text{C}$:
- Theoretische Carnot-limiet: $COP_{Carnot,kamer} = \frac{292{,}65}{19{,}5} \approx 15{,}01$
- Gemeten cyclus-COP van het gehele systeem (inclusief pomp & defrost): `2,80`
- Second-Law Efficiency: $\eta_{II,kamer} = \frac{2{,}80}{15{,}01} \approx \mathbf{18{,}7\%}$

---

## 3. Experimenteel Ontwerp & Statistische Validatie (Home Assistant PoC)
Om vertekening door overgangseffecten en meetruis te voorkomen, gebruikt het leermodel een **cyclus-geblokkeerde online leermethode**:

1. **Integratie over volledige cycli (van defrost tot defrost):**
   - De flow (`manualIpwm`) wordt gedurende een volledige verwarmingscyclus strikt constant gehouden.
   - Zowel de geleverde thermische energie ($Q_{geleverd}$) als de Carnot-genormaliseerde energie ($E_{Carnot,kamer}$) worden over de gehele cyclus geïntegreerd, inclusief de ontdooifase.
   - Cycli korter dan `35 min` of langer dan `180 min` worden uitgesloten om opstart- en uitschakelbias te voorkomen.
2. **Vermogensklassen (Buckets):**
   - De geleerde statistieken worden gescheiden in twee gemiddelde warmtevraag-regimes: **Laag/Midden ($\le 3{,}2\text{ kW}$)** en **Hoog ($> 3{,}2\text{ kW}$)**.
3. **Verkenning versus Benutting (Exploration vs. Exploitation):**
   - Test discrete flow-stappen (bijv. `608 iPWM` $\approx 435\text{ L/u}$, `555 iPWM` $\approx 525\text{ L/u}$, `500 iPWM` $\approx 605\text{ L/u}$) door uitsluitend bij de start van een defrost te wisselen.
4. **Online Variantie & Significantietoets (Welford's Algoritme):**
   - Houdt per klasse en flow-stap het aantal cycli ($n$), het lopend gemiddelde ($\mu$) en de kwadratensom van afwijkingen ($M_2$) bij.
   - Standaardfout:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - Een hogere flow-stap wordt pas als winnaar gekozen zodra $n \ge 4$ en het gemiddelde $\eta_{II,kamer}$ de baseline overtreft met minimaal $1{,}5 \times \sqrt{SE_{baseline}^2 + SE_{kandidaat}^2}$.

---

## 4. Vervolgstappen voor Firmware-integratie
- Meerdaagse baseline- en A/B-cyclusdatasets verzamelen over verschillende buitentemperatuurregimes en afgiftesystemen (radiatoren, VVW en mix).
- Evalueren of een lichtgewicht variant van cyclus-geïntegreerde $\eta_{II,kamer}$-tracking op termijn de flow-regelparameters van OpenQuatt (`flowKp`, `flowKi` of dynamische flowcurves) kan aanvullen of automatisch kan afstemmen.
