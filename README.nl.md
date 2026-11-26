[🇬🇧 English](README.md) | 🇳🇱 Nederlands

# Zelflerende Flow-optimalisatie voor OpenQuatt

## Inleiding
Een warmtepomp geeft zijn warmte af aan het water dat door je radiatoren of vloerverwarming stroomt. Hoe snel dat water stroomt (de **flow**, in liter per uur), bepaalt direct hoe warm het water wordt en hoeveel stroom de installatie verbruikt:
- **Stroomt het water langzamer?** Dan neemt elke liter water meer warmte op ($\Delta T$ wordt groter). Daardoor stijgt de aanvoertemperatuur ($T_{aanvoer}$) en loopt de druk in de warmtepomp op — zelfs op de laagste stand (30 Hz).
- **Stroomt het water sneller?** Dan blijft de aanvoertemperatuur ($T_{aanvoer}$) lager, maar verbruikt de waterpomp zelf meer stroom. Ook komt het water warmer terug bij de warmtepomp, waardoor die zijn warmte minder goed kwijt kan.

Elk huis is anders. Het ene huis heeft alleen radiatoren, het andere vloerverwarming, en weer een ander huis heeft allebei. Daarom werkt één vaste instelling niet voor iedereen het beste.

Dit project laat Home Assistant **zelf meten en leren** welke watersnelheid in jouw huis het meeste rendement oplevert. Het systeem test verschillende standen steeds over een volledige werkcyclus en kiest pas een nieuwe stand als de metingen met voldoende zekerheid laten zien dat die zuiniger is.

---

## 1. Probleemstelling & Motivatie (Radiatoren, VVW en Mix)
Bij watergedragen warmtepompsystemen is het kiezen van de optimale waterflow een afweging tussen warmteoverdracht en pompenergie. Omdat de warmteafgifte wordt bepaald door de gemiddelde watertemperatuur ($T_{gem} = \frac{T_{aanvoer} + T_{retour}}{2}$), dwingt een hogere $\Delta T$ ($T_{aanvoer} - T_{retour}$) altijd een hogere aanvoertemperatuur ($T_{aanvoer}$) af.

Het optimale $\Delta T$-venster en hydraulische gedrag verschuiven per afgiftesysteem:

| Systeemtype | Typische optimale $\Delta T$ | Thermisch & hydraulisch gedrag |
| :--- | :--- | :--- |
| **Alleen radiatoren** | **5,0 – 6,5 K** | Radiatoren hebben relatief weinig waterinhoud. Bij te lage flow (hoge $\Delta T$) neemt de turbulentie in de platenwisselaar en radiatoren af; bij te hoge flow stijgt het pompverbruik en kan het koudemiddel in de wisselaar minder goed afkoelen. |
| **Alleen vloerverwarming (VVW)** | **3,0 – 5,0 K** | VVW heeft een groot afgifteoppervlak en grote thermische massa, waardoor $T_{aanvoer}$ al laag ligt (28–35 °C). Een hogere flow houdt de vloer gelijkmatiger warm en verlaagt de condensordruk, zolang het kubische pompverbruik beperkt blijft. |
| **Mix (radiatoren + VVW)** | **4,5 – 6,0 K** *(primair circuit)* | De radiatoren bepalen hier de benodigde aanvoertemperatuur. Een mengverdeler met eigen pomp trekt de primaire en secundaire flow uit elkaar, waardoor $T_{retour}$ snel oploopt als het primaire debiet te hoog wordt. |

Omdat leidingweerstand, pompcurve en afgifte-oppervlak per woning verschillen, is één vast $\Delta T$-getal in firmware nooit voor elke woning ideaal.

---

## 2. Meetmethode: Zelf-ijkend Kamerrendement (~100%-schaal)
Om verschillende pompsnelheden eerlijk te vergelijken bij wisselend weer én verschillende compressortoerentallen — zonder handmatige correctiefactoren per woningtype — gebruikt dit model een **zelf-ijkende rendementsindex**:

$$\text{Referentie-COP}_{kamer}(Hz) = \frac{T_{kamer} + 273{,}15}{T_{kamer} - T_{buiten,ODU}} \cdot f_{ODU}$$

$$\text{Kamerrendement} = \frac{\text{COP}_{systeem} / \text{Referentie-COP}_{kamer}(Hz)}{K_{baseline}} \times 100\%$$

Waarbij:
- $T_{kamer}$: de continu gemeten ruimtetemperatuur (geen vast setpoint, zodat opwarmen of zoninstraling eerlijk meeweegt).
- $T_{buiten,ODU}$: de fysiek gemeten luchttemperatuur bij de buitenunit.
- $f_{ODU}$: de geijkte efficiëntiefactor van de buitenunit per compressorfrequentie (`30–90 Hz`).
- $K_{baseline}$: wordt tijdens de eerste geldige cyclus automatisch vastgelegd, zodat de baseline van elk huis (ongeacht radiatoren, VVW of mix) vanzelf op **100,0%** start.

### Waarom deze maatstaf?
1. **Geen handmatige instellingen per woning:** Of je nu radiatoren, vloerverwarming of een mix hebt, het systeem ijkt zichzelf op de eerste baseline-cyclus.
2. **Onafhankelijk van weer en toerental:** Zowel schommelingen in $T_{buiten,ODU}$ als het lagere compressorrendement op hoge frequenties (80–90 Hz) worden genormaliseerd.
3. **Meet de hele keten in één cijfer:** Omdat $T_{aanvoer}$ niet in de referentie zit, verschuift de lat niet mee als een hogere flow $T_{aanvoer}$ doet dalen. Elke nettowinst in warmteoverdracht — minus het extra pompvermogen ($P_{pomp}$) en ontdooiverliezen — wordt direct zichtbaar als een score boven `100%`.

---

## 3. Experimenteel Ontwerp & Statistische Validatie (Home Assistant PoC)
Om vertekening door overgangseffecten en meetruis te voorkomen, gebruikt het leermodel een **cyclus-geblokkeerde online leermethode**:

1. **Integratie over volledige cycli (defrost-to-defrost of 90-minuten venster):**
   - De flow (`manualIpwm`) wordt gedurende een volledige cyclus strikt constant gehouden.
   - Bij vorst wisselt het model bij de start van elke ontdooicyclus (*defrost*). Bij mild weer (lagere compressorstanden van `30–55 Hz` zonder defrosts) sluit het model na **90 minuten** onafgebroken verwarmen automatisch een meetblok af.
   - Cycli korter dan `35 min` of langer dan `180 min` worden uitgesloten om opstart- en uitschakelbias te voorkomen.
2. **Vermogensklassen (Buckets):**
   - De geleerde statistieken worden gescheiden in twee gemiddelde warmtevraag-regimes: **Laag/Midden ($\le 3{,}2\text{ kW}$)** en **Hoog ($> 3{,}2\text{ kW}$)**.
3. **Verkenning versus Benutting (Exploration vs. Exploitation):**
   - Test discrete flow-stappen (bijv. `608 iPWM` $\approx 435\text{ L/u}$, `555 iPWM` $\approx 525\text{ L/u}$, `500 iPWM` $\approx 605\text{ L/u}$).
4. **Online Variantie & Significantietoets (Welford's Algoritme):**
   - Houdt per klasse en flow-stap het aantal cycli ($n$), het lopend gemiddelde ($\mu$) en de kwadratensom van afwijkingen ($M_2$) bij.
   - Standaardfout:
     $$SE = \sqrt{\frac{M_2}{n(n - 1)}}$$
   - Een hogere flow-stap wordt pas als winnaar gekozen zodra $n \ge 4$ en het gemiddelde kamerrendement de baseline overtreft met minimaal $1{,}5 \times \sqrt{SE_{baseline}^2 + SE_{kandidaat}^2}$.

---

## 4. Installatie (Home Assistant)
1. **Backend (Package):** Plaats alleen `packages/openquatt_flow_learning.yaml` in je Home Assistant `/config/packages/` map (zorg dat `packages: !include_dir_named packages` in je `configuration.yaml` staat) en herstart Home Assistant.
2. **Dashboardkaart:** Open je dashboard in Home Assistant → **Bewerken** → **+ Kaart toevoegen** → **Handmatig** en plak de inhoud van `dashboard.yaml` erin. *(Plaats `dashboard.yaml` niet in `/config/packages/`).*
3. **Nulmeting & Start:** Laat `input_boolean.oq_flow_learning_actief` eerst enkele cycli uit staan om de automatische nulmeting (`100,0%`) vast te leggen en schakel hem daarna in om automatisch te testen.
