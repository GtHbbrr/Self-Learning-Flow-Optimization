[🇬🇧 English](README.md) | 🇳🇱 Nederlands

# Zelflerende Flow-optimalisatie voor OpenQuatt

## Inleiding
Een warmtepomp geeft zijn warmte af aan het water dat door je radiatoren of vloerverwarming stroomt. Hoe snel dat water stroomt (de **flow**, in liter per uur), bepaalt direct hoe warm het water wordt en hoeveel stroom de installatie verbruikt:
- **Stroomt het water langzamer?** Dan neemt dezelfde hoeveelheid warmte per liter water meer toe ($\Delta T$ wordt groter). Daardoor stijgt de aanvoertemperatuur ($T_{aanvoer}$) en loopt de druk in de compressor op — zelfs op het laagste toerental (30 Hz).
- **Stroomt het water sneller?** Dan blijft $T_{aanvoer}$ lager, maar verbruikt de waterpomp zelf meer stroom en komt het water warmer terug bij de warmtepomp, waardoor het koudemiddel minder goed onderkoelt.

Elk huis is anders. Het ene huis heeft alleen radiatoren, het andere vloerverwarming, en weer een ander huis heeft allebei. Daarom werkt één vaste instelling niet voor iedereen het beste.

Dit project laat Home Assistant **zelf meten en leren** welke watersnelheid in jouw huis het meeste rendement oplevert. Het systeem test verschillende standen steeds over een volledige werkcyclus en kiest pas een nieuwe stand als de metingen met voldoende zekerheid laten zien dat die zuiniger is.

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

## 2. Meetmethode: Kamerrendement
Om verschillende pompsnelheden eerlijk te vergelijken bij wisselend weer, meten we het **kamerrendement**. Dit getal laat zien hoeveel van het maximaal haalbare rendement je warmtepomp, waterpomp en radiatoren samen behalen om de kamer warm te houden:

$$\text{Kamerrendement} = \frac{\text{COP}_{systeem}}{\text{COP}_{Carnot,kamer}} = \frac{Q_{geleverd} / E_{totaal,in}}{\frac{T_{kamer} + 273{,}15}{T_{kamer} - T_{buiten,ODU}}}$$

### Waarom kamerrendement in plaats van ruwe COP of Carnot op aanvoertemperatuur?
1. **Onafhankelijk van het weer:** Door te vergelijken met het theoretisch maximum tussen buiten en binnen ($COP_{Carnot,kamer}$), verstoren wisselingen in de buitentemperatuur ($T_{buiten,ODU}$) de meting niet.
2. **Meet de hele keten in één cijfer:** Formules op basis van de watertemperatuur ($T_{aanvoer}$) verlagen de lat zodra een hogere flow $T_{aanvoer}$ doet dalen. Het kamerrendement kijkt puur naar het eindresultaat — warmte van buiten naar de woonkamer brengen — en neemt alles mee:
   - Warmteoverdracht in de condensor (*pinch point* & *subcooling*),
   - Warmteoverdracht in radiatoren en/of vloerverwarming,
   - Het stroomverbruik van de waterpomp ($P_{pomp}$ zit in $E_{totaal,in}$),
   - Energieverlies tijdens ontdooicycli (*defrosts*).

### Rekenvoorbeeld
Bij $T_{buiten,ODU} = 0{,}0\text{ }^\circ\text{C}$ en $T_{kamer} = 19{,}5\text{ }^\circ\text{C}$:
- Maximaal haalbaar rendement (Carnot-limiet): $COP_{Carnot,kamer} = \frac{292{,}65}{19{,}5} \approx 15{,}01$
- Gemeten rendement over de hele cyclus (inclusief pomp & defrost): `2,80`
- Kamerrendement: $\frac{2{,}80}{15{,}01} \approx \mathbf{18{,}7\%}$

---

## 3. Experimenteel Ontwerp & Statistische Validatie (Home Assistant PoC)
Om vertekening door overgangseffecten en meetruis te voorkomen, gebruikt het leermodel een **cyclus-geblokkeerde online leermethode**:

1. **Integratie over volledige cycli (van defrost tot defrost):**
   - De flow (`manualIpwm`) wordt gedurende een volledige verwarmingscyclus strikt constant gehouden.
   - Zowel de geleverde warmte ($Q_{geleverd}$) als de benodigde referentie-energie ($E_{Carnot,kamer}$) worden over de gehele cyclus opgeteld, inclusief de ontdooifase.
   - Cycli korter dan `35 min` of langer dan `180 min` worden uitgesloten om opstart- en uitschakelbias te voorkomen.
2. **Vermogensklassen (Buckets):**
   - De geleerde statistieken worden gescheiden in twee gemiddelde warmtevraag-regimes: **Laag/Midden ($\le 3{,}2\text{ kW}$)** en **Hoog ($> 3{,}2\text{ kW}$)**.
3. **Verkenning versus Benutting (Exploration vs. Exploitation):**
   - Test discrete flow-stappen (bijv. `608 iPWM` $\approx 435\text{ L/u}$, `555 iPWM` $\approx 525\text{ L/u}$, `500 iPWM` $\approx 605\text{ L/u}$) door uitsluitend bij de start van een defrost te wisselen.
4. **Online Variantie & Significantietoets (Welford's Algoritme):**
   - Houdt per klasse en flow-stap het aantal cycli ($n$), het lopend gemiddelde ($\mu$) en de kwadratensom van afwijkingen ($M_2$) bij.
   - Standaardfout:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - Een hogere flow-stap wordt pas als winnaar gekozen zodra $n \ge 4$ en het gemiddelde kamerrendement de baseline overtreft met minimaal $1{,}5 \times \sqrt{SE_{baseline}^2 + SE_{kandidaat}^2}$.

---

## 4. Vervolgstappen voor Firmware-integratie
- Meerdaagse baseline- en A/B-cyclusdatasets verzamelen over verschillende buitentemperatuurregimes en afgiftesystemen (radiatoren, VVW en mix).
- Evalueren of een lichtgewicht variant van cyclus-geïntegreerde kamerrendement-tracking op termijn de flow-regelparameters van OpenQuatt (`flowKp`, `flowKi` of dynamische flowcurves) kan aanvullen of automatisch kan afstemmen.
