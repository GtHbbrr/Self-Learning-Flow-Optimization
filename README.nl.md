# Zelflerende Flow-optimalisatie voor OpenQuatt (`η_II,kamer`)

## 1. Probleemstelling & Motivatie
Bij watergedragen warmtepompsystemen (en in het bijzonder bij installaties met radiatoren) is het kiezen van de optimale waterflow een lastige afweging:
- **Te lage flow (`~435 L/u`):** Vergroot de $\Delta T$ over de radiatoren (7–9 K), wat een hogere aanvoertemperatuur ($T_{aanvoer}$) en hogere condensordruk ($P_{cond}$) afdwingt. Hierdoor daalt het isentropisch en volumetrisch rendement van de compressor.
- **Te hoge flow (`>700 L/u`):** Verhoogt het kubische elektriciteitsverbruik van de circulatiepomp ($P_{pomp}$), verhoogt de retourtemperatuur ($T_{retour}$), vermindert de onderkoeling (*subcooling*) van het vloeibare koudemiddel in de platenwisselaar en kan stromingsgeluid veroorzaken.

Statische stooklijnen of zuivere $\Delta T$-gebaseerde PID-regelaars houden geen rekening met het installatiespecifieke radiatoroppervlak, de hydraulische weerstand of het werkelijke systeemrendement over volledige ontdooicycli (*defrosts*).

---

## 2. Doelfunctie: Second-Law Efficiency op kamerniveau ($\eta_{II,kamer}$)
Om verschillende flows eerlijk te vergelijken bij wisselende buitentemperaturen en bedrijfsomstandigheden, optimaliseert deze Proof of Concept (PoC) op de **Second-Law (exergie) efficiëntie tussen de buitenlucht en de kamertemperatuur**:

$$\eta_{II,kamer} = \frac{\text{COP}_{systeem}}{\text{COP}_{Carnot,kamer}} = \frac{Q_{geleverd} / E_{totaal,in}}{\frac{T_{kamer} + 273{,}15}{T_{kamer} - T_{buiten,ODU}}}$$

### Waarom $\eta_{II,kamer}$ in plaats van ruwe COP of Carnot op aanvoertemperatuur?
1. **Weersgenormaliseerd:** Delen door $COP_{Carnot,kamer}$ haalt het primaire effect van schommelingen in de buitentemperatuur ($T_{buiten,ODU}$) uit de vergelijking.
2. **Omvat de volledige thermische keten:** In tegenstelling tot Carnot-formules op basis van $T_{aanvoer}$ (die de referentielat verlagen zodra een hogere flow $T_{aanvoer}$ doet dalen), beoordeelt $\eta_{II,kamer}$ het **totaalrendement** van warmtetransport van buitenlucht naar de verwarmde kamer:
   - Warmteoverdracht in de condensor (*pinch point* & *subcooling*),
   - Warmteoverdracht in de radiatoren (LMTD bij de gekozen flow),
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
- Meerdaagse baseline- en A/B-cyclusdatasets verzamelen over verschillende buitentemperatuurregimes.
- Evalueren of een lichtgewicht variant van cyclus-geïntegreerde $\eta_{II,kamer}$-tracking op termijn de flow-regelparameters van OpenQuatt (`flowKp`, `flowKi` of dynamische flowcurves) kan aanvullen of automatisch kan afstemmen.
