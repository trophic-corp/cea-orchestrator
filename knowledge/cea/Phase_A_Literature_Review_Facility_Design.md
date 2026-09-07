**PREMIUM INDIAN CEA EQUIPMENT MANUFACTURING**

**R&D Strategy Report --- Phase A**

*Scientific Literature Review & Optimal Mid-Tech CEA Research Facility
Design*

Prepared for: Ooty R&D Facility (Microgreens · Aquascaping Plants ·
Saffron)

Manufacturing base: Coimbatore, Tamil Nadu, India

*Document 1 of a phased report series (Objective 1 of 6)*

Note on Scope

This document is Phase A of a six-part strategic report. It covers the
Scientific Literature Review and the R&D Facility Design (Objective 1).
Subsequent phases will cover Global Technology Benchmarking, Product
Portfolio Development, Manufacturing Feasibility in Coimbatore, the Bill
of Materials/Supplier Catalog, and the Manufacturing Roadmap --- each as
its own linked deliverable, before final assembly into the complete
report.

All recommendations below are grounded in peer-reviewed research,
university publications, and CEA engineering literature published
primarily within the last decade, supplemented by recent industry/trade
sources where academic literature is sparse (flagged explicitly, e.g.
aquatic ornamental plant nursery practice). Every subsystem
recommendation states the evidence strength as Strong, Moderate, or
Emerging.

1\. Scientific Literature Review

1.1 Lighting --- Spectrum, Intensity, and Photoperiod

Light quality, intensity, and photoperiod are the dominant levers for
both yield and phytochemical quality in controlled-environment leafy
crops. Across multiple controlled studies on Brassicaceae and other
microgreens, blue-enriched and red:blue combined spectra consistently
outperform white or single-wavelength light for driving both biomass
accumulation and secondary-metabolite synthesis (ascorbic acid,
phenolics, anthocyanins, carotenoids). One study comparing white, red,
and blue LED treatments found the highest ascorbic acid, total phenolic
content, and antioxidant capacity under blue-spectrum lighting, while
fresh-weight yield in amaranth and turnip greens was maximized under
blue relative to red or white light.

A recent proteomic study on cress microgreens found that shifting the
blue:green:red:far-red ratio (comparing 13:15:61:11 to 24:12:56:8) did
not change total yield or dry matter but significantly increased
anthocyanin (+77%) and phenolic index (+52%) under the blue-enriched,
far-red-reduced spectrum --- indicating spectrum can be tuned for
nutritional quality independent of yield. A review in Plant Growth
Regulation further establishes that photosynthetic photon flux densities
(PPFD) above roughly 210 µmol m⁻² s⁻¹, especially combined with far-red
or UV-A, tend to induce photoinhibition and reduce yield, while PPFD
below 100 µmol m⁻² s⁻¹ suppresses both biomass and phytochemical
accumulation --- establishing a practical working band of roughly
150--210 µmol m⁻² s⁻¹ for microgreens.

Photoperiod also has an independent, species-dependent effect. A
controlled study on four Brassicaceae microgreen genotypes (arugula,
broccoli, mizuna, radish) found that continuous lighting (24h) increased
both yield and phytochemical content relative to a 16-hour photoperiod,
with effects more pronounced under LED than fluorescent light --- though
continuous lighting increases energy draw and must be weighed against
the marginal quality gain.

***Evidence strength:** Strong (multiple independent, peer-reviewed,
dose-response studies converge on similar spectral/PPFD ranges).*

1.2 Vapor Pressure Deficit (VPD) and Humidity Control

VPD --- the deficit between the saturated vapor pressure at leaf
temperature and the actual vapor pressure of the surrounding air --- is
now treated in CEA engineering literature as a more actionable control
variable than relative humidity alone, because it directly governs
stomatal aperture, transpiration-driven nutrient transport, and disease
pressure. A 2025 study using a NeuralProphet-based forecasting model for
vertical farm VPD demonstrated that predictive climate control (rather
than simple setpoint control) meaningfully improves resource-use
efficiency and reduces the risk of both under- and over-transpiration.
Systematic reviews of greenhouse humidity control confirm that
fuzzy-logic and model-predictive approaches outperform simple on/off
dehumidification for maintaining stable VPD, which is particularly
relevant for crops sensitive to both under-transpiration (low VPD →
nutrient deficiency, disease) and over-transpiration (high VPD →
wilting, tip-burn).

***Evidence strength:** Strong (well-established plant physiology) for
the underlying VPD mechanism; Moderate for predictive/ML-based VPD
control specifically.*

1.3 CO₂ Enrichment

CO₂ enrichment is one of the most consistently validated interventions
in CEA. In closed plant factories with artificial lighting (PFAL),
studies on lettuce found that doubling (approx. 800 µmol mol⁻¹) to
quadrupling ambient CO₂ significantly increased biomass, growth rate,
and light-use efficiency, with the broader literature converging on an
optimal range of roughly 1,000--1,500 µmol mol⁻¹ for closed PFAL systems
(versus \~800 µmol mol⁻¹ typically used in ventilated greenhouses). A
review in Frontiers in Plant Science found that moderate CO₂ elevation
(550--650 µmol mol⁻¹) improves yield of C3 crops (which include nearly
all leafy greens, microgreens, and saffron) by an average of 18%
relative to ambient CO₂, with C3 species responding far more strongly
than C4 or CAM plants. A gas-exchange study on tomato additionally found
that raising CO₂ from 400 to 1,000 µmol mol⁻¹ increased net
photosynthesis by 51%, decreased transpiration by 5--8%, and improved
photosynthetic water-use efficiency by 60% --- meaning CO₂ enrichment
also reduces irrigation/dehumidification load, a secondary energy
benefit relevant to a sealed room design.

***Evidence strength:** Strong (large, converging body of controlled
experimental evidence across multiple crop families).*

1.4 Airflow and Canopy-Level Air Movement

Airflow design directly affects the leaf boundary layer, which governs
gas exchange and transpiration uniformity. Computational fluid dynamics
(CFD) studies on plant factories consistently show that horizontal air
velocities at the canopy surface in the range of roughly 0.2--0.5 m/s
(with an upper practical bound near 1.3 m/s before
mechanical/desiccation stress) are needed to prevent the stagnant
boundary layer responsible for physiological disorders such as lettuce
tip-burn (a calcium-transport disorder caused by insufficient
transpiration in low-airflow zones). A CFD-optimized multi-duct design
achieved 0.42 m/s average velocity with a 44% coefficient of variation,
while a later commercial-scale CFD study found that distributing air
inlets along the full length of a growing shelf --- rather than from a
single end --- reduced the coefficient of variation from \~33% to \~10%,
demonstrating that inlet placement geometry matters as much as total fan
capacity.

***Evidence strength:** Strong (multiple validated CFD studies with
consistent physiological outcome measures).*

1.5 Energy Efficiency of Lighting and HVAC Interaction

Because essentially all electrical energy delivered to LED fixtures that
is not converted to usable photosynthetically active radiation (PAR) is
ultimately dissipated as heat that the HVAC system must remove, lighting
and climate-control subsystems must be engineered jointly rather than
independently. A 2024 benchmarking review found current specific energy
consumption for vertical-farmed lettuce of 10--18 kWh/kg, against a
theoretical technical benchmark of 3.1--7.4 kWh/kg --- indicating
substantial efficiency headroom remains in commercial systems. The same
review notes electricity-to-PAR conversion efficiency of roughly 50% for
modern LEDs versus 33% for legacy HPS lighting. Close-canopy lighting
research from Purdue University demonstrated that reducing the vertical
distance between LED fixtures and the canopy --- while dimming to hold
PPFD constant --- cuts both direct lighting energy and indirect HVAC
cooling load, because photons that would otherwise strike walls, aisles,
and empty positions (pure waste) are captured more efficiently. A
systematic review of 52 vertical-farming energy studies (2014--2024)
confirms that combining high-efficacy LEDs, smart/predictive HVAC
control, and IoT-based irrigation delivers the largest realistic energy
reductions, though savings remain highly dependent on climate, layout,
and crop mix.

***Evidence strength:** Strong for the lighting-heat-HVAC coupling
mechanism; Moderate-to-Strong for the magnitude of achievable savings,
which is facility-specific.*

1.6 Automated Fertigation, pH, and EC Control

A 2025 scoping review of 89 studies on automated hydroponic nutrient
dosing found a steady rise in research output since 2015, with pH (n=70
studies), EC (n=36), and nutrient solution volume (n=42) as the dominant
control variables, and feedback-loop and predictive-analytics dosing
frameworks equally represented. Extreme pH (below 5 or above 8) was
shown to reduce nutrient solubility and stunt growth, with an optimal
range of roughly pH 5.6--6.2 identified for several crop types --- a
range broadly consistent with saffron, leafy greens, and most
aquatic/emergent ornamental species. Engineering studies combining PLC +
HMI control with peristaltic/proportional dosing pumps and inline flow
sensors demonstrated pH/EC control errors reduced to within 3% of
target, alongside a 50% reduction in irrigation cycle time relative to
manual dosing --- supporting PLC-based fertigation as the appropriate
mid-tech automation tier for a premium product line, ahead of fully
manual dosing but short of the fully autonomous ML-optimized systems
(e.g., the multi-fertilizer-source \'OptiDose\' framework) still
confined to research settings.

***Evidence strength:** Strong for pH/EC feedback control fundamentals;
Emerging for multi-ion optimal-control (ML-based) dosing.*

1.7 IoT Sensor Networks and Low-Cost Imaging/Phenotyping

A 2026 review synthesizing over 130 studies (2020--2025) on
smart-greenhouse AI applications found that sensor data in commercial
smart-greenhouse systems clusters into two families: spatial imaging
(RGB, multispectral, hyperspectral, thermal) for growth/disease/yield
assessment, and time-series environmental sensors (temperature,
humidity, CO₂, light, root-zone EC/pH) used for forecasting and control
optimization. A separate 114-study review found that multi-sensor
integration improves monitoring precision and system resilience but
introduces calibration and interoperability challenges, and that
intelligent (fuzzy-logic, model-predictive, or reinforcement-learning)
control improves energy and water efficiency but requires larger,
cleaner datasets than most small facilities generate on their own. On
the low-cost end, demonstrated ESP8266/DHT-class sensor networks
integrated with cloud dashboards (e.g., Firebase) have been implemented
for well under US \$200 per monitoring node, and open-hardware
environmental sensor platforms combined with 3D-printed imaging robots
have been published specifically for phenotyping in space-constrained
research settings --- directly relevant to a compact Ooty research room.

***Evidence strength:** Strong for the value of multi-sensor monitoring;
Moderate for AI/ML-driven autonomous control at small scale, which is
still an active research area.*

1.8 Saffron (Crocus sativus) --- Controlled-Environment Requirements

Saffron corm physiology has been characterized with unusual precision
because of the crop\'s high commercial value. Controlled-temperature
studies establish a clear two-stage thermal protocol: corms should be
germinated/stored at a warmer temperature (approximately 23--30°C)
before being transitioned to a cooler flower-induction phase
(approximately 15--16°C), with 16°C identified as the optimum for flower
formation provided prior warm-phase germination. A separate greenhouse
study found that even lower day/night regimes (15°C/6°C) represent
optimal corm development conditions, and that cooling of the nutrient
solution specifically increased flower number, stigma weight, and the
concentration of the commercially important compounds crocin,
picrocrocin, and safranal. Because saffron flower induction depends on a
temperature drop relative to a warmer prior phase, a single research
room that can be run through a programmable two-zone or two-season
temperature protocol is well suited to Ooty\'s naturally cool ambient
climate (see Section 2.2), which can supply a substantial share of the
cooling load passively for much of the year --- a genuine site-specific
advantage over saffron CEA operations in warmer parts of India.

High-temperature stress research (relevant as a climate-risk mitigation
reference, given warming trends even at altitude) found that combining
Trichoderma harzianum bio-inoculant treatment with nano-TiO₂ application
under stress conditions of 25°C/15°C day/night (10°C above optimum)
produced measurably larger, more vigorous cormlets --- a secondary R&D
avenue for corm-multiplication research once the base facility is
operational.

***Evidence strength:** Strong for temperature/flowering relationships
(multiple independent replicated studies); Emerging for
nanoparticle/bio-inoculant cormlet enhancement techniques.*

1.9 Aquatic and Aquascaping Plant Cultivation

Commercial aquatic-plant nurseries overwhelmingly propagate stock in the
emersed form (roots submerged in a moist nutrient substrate, foliage
grown in open air) rather than fully submersed, because emersed
cultivation is faster, cheaper, less prone to algae and snail
contamination, and produces sturdier plants that ship and transplant
more reliably --- after which the plant is transitioned to its submersed
morphology by the end retailer or aquascaper. Species such as Anubias,
Cryptocoryne, Bucephalandra, Bolbitis, Hygrophila, Rotala, and the
carpeting genera (Hemianthus, Micranthemum, Eleocharis, Marsilea) are
all documented as suitable for emersed high-humidity propagation.
Tissue-culture-produced aquatic plants are grown emersed under sterile
in-vitro conditions specifically to guarantee a pest- and algae-free
starting stock, which commands a price premium in the aquascaping trade.
It should be noted plainly that the literature on commercial emersed
aquatic-plant propagation is dominated by industry/trade sources and
hobbyist nursery documentation rather than peer-reviewed horticultural
science --- the underlying plant-physiology principles (humidity
dependence, substrate nutrition, LED lighting response) are well studied
in general CEA literature, but species-specific, replicated agronomic
trials for ornamental aquatics are largely absent from the academic
record. This is itself a notable white-space opportunity: a premium
Indian brand publishing its own trial data on emersed propagation
protocols (light recipes, humidity ramp schedules, substrate
formulations) would be establishing genuinely novel IP in an
under-researched niche.

***Evidence strength:** Moderate-to-Emerging (strong physiological
rationale from adjacent CEA literature; direct species-level evidence is
largely industry/grey literature, not peer-reviewed).*

2\. R&D Facility Design --- Ooty Research Room

2.1 Site Context: Ooty Climate as a Design Input

Ooty (Udhagamandalam) sits at approximately 2,240 m altitude in the
Nilgiri Hills, with a Köppen Cwb/Cfb subtropical highland climate.
Ambient temperatures typically range from roughly 5--25°C across the
year, with daytime highs near 25--30°C in the warmest months
(March--May) and nights commonly falling to 5--13°C, especially
November--February. Relative humidity is highest during the
June--September southwest monsoon (annual rainfall approximately
1,500--1,900 mm) and lowest in the dry season around March. This is
climatically distinct from nearly every other Indian CEA installation,
most of which are engineered around aggressive cooling loads for a hot,
humid, or semi-arid ambient climate.

This has two direct design implications. First, the facility\'s dominant
HVAC challenge for most of the year is likely to be gentle heating and
humidity/VPD management rather than heavy refrigeration-grade cooling
--- the opposite of the typical Indian CEA HVAC sizing problem, and a
meaningful reduction in compressor-sizing and running-cost relative to a
Coimbatore-plains facility. Second, Ooty\'s naturally cool ambient
temperature is directly useful for saffron\'s cool-phase
flower-induction requirement (Section 1.8): the room\'s cooling
subsystem can lean on outside-air economizer cooling for much of the
flower-induction window rather than running mechanical refrigeration
continuously, which is not an option available to saffron CEA projects
sited on the Tamil Nadu plains or in most of India. This is a
legitimate, evidence-grounded differentiator worth foregrounding in the
company\'s technical marketing.

2.2 Room Envelope and Insulation

Recommended room size: 12×12×12 ft (mid-point of the specified 10×10×12
to 15×15×12 ft range), giving approximately 144 sq ft of floor area and
sufficient height for a 2-tier or partial 3-tier rack system with
service clearance. Construction should use pre-fabricated insulated
sandwich panels (PUF or PIR core, steel or aluminum face sheets) rather
than conventional masonry --- this is standard practice for cold-room
and plant-factory envelopes, with 80--100 mm PUF-core panels achieving
U-values around 0.20--0.25 W/m²K and typical thermal conductivities near
0.022 W/m·K. Sandwich-panel construction cuts build time by roughly
30--50% versus masonry, is easier to keep hygienic (a single flat
cleanable interior face --- relevant for microgreens food-safety and
tissue-culture cleanliness), and can be relocated or reconfigured if the
R&D program\'s footprint needs change. PIR core (over EPS) is
recommended for its better fire classification given the presence of
electrical/lighting load inside the envelope. All panel joints should be
thermally broken and sealed to avoid condensation bridging, which is a
known failure point in insulated-panel plant-factory rooms.

2.3 Zone Allocation for the Three-Crop Program

Because microgreens, aquascaping plants, and saffron have materially
different environmental setpoints (saffron in particular needs a
distinct, colder induction phase), the room should be designed as a
single insulated envelope subdivided into two or three semi-independent
microclimate zones using internal curtain partitions or a modular
rack-level enclosure, each with its own airflow and (where needed)
temperature control, rather than one shared climate for the whole room.
A practical allocation within the 144 sq ft footprint:

-   Zone A --- Microgreens (30--40% of floor area): multi-tier NFT/tray
    racks, 16--20h photoperiod, PPFD 150--210 µmol m⁻² s⁻¹, temperature
    20--24°C, moderate VPD.

-   Zone B --- Aquascaping/aquatic plants (30--40% of floor area):
    shallow emersed propagation trays or paludarium-style benches, high
    humidity (\>80% RH) with a humidity dome/misting subsystem,
    warm-white/full-spectrum LED at lower intensity than the microgreen
    zone, 12--16h photoperiod.

-   Zone C --- Saffron (10--20% of floor area, expandable seasonally): a
    temperature-programmable enclosed sub-chamber run through a
    two-phase protocol --- warm corm-storage/germination phase
    (23--30°C) transitioning to a cool flower-induction phase (approx.
    15--16°C, leaning on Ooty ambient/economizer cooling),
    low-to-moderate light during induction, controlled dry-to-moist
    irrigation timing.

This zoned approach also directly supports the future-expansion crops
named in the brief (leafy greens, herbs, medicinal plants,
tissue-culture hardening): each is simply a fourth or fifth zone profile
added to the same control architecture, rather than a different
facility.

2.4 Subsystem Architecture

The table below summarizes the recommended subsystem architecture for a
premium mid-tech version of this room --- deliberately positioned above
basic hobbyist/DIY builds (fixed-spectrum lights, manual dosing, no data
logging) and below full industrial PFAL-grade automation (multi-ion
optimal-control dosing, full AI climate control, robotic handling),
which is appropriate both for a first-generation R&D room and for the
mid-tech commercial product tier the company is targeting.

  ---------------------------------- ----------------------------------------------------------------- ------------------------------------------------------------------------------ ------------------------------------------------------------------------------------------------------------------
  **Subsystem**                      **Why Needed**                                                    **State-of-the-Art Approach**                                                  **Recommended Mid-Tech Solution**
  Lighting                           Drives yield and phytochemical quality; largest energy load       Dynamic multi-channel tunable spectrum with per-zone dimming (Sec. 1.1, 1.5)   Dimmable RB(+W/FR) LED bars, \~150--210 µmol m⁻²s⁻¹, close-canopy mounting, zone-specific photoperiod control
  Environmental control (T/RH/VPD)   Governs transpiration, disease pressure, tip-burn risk            Predictive/model-based VPD control (Sec. 1.2)                                  Setpoint VPD control via networked T/RH sensors driving heater, humidifier/mister, and dehumidifier per zone
  CO2 enrichment                     18--40%+ yield gains in C3 crops in closed rooms (Sec. 1.3)       Null-balance CO2 dosing to 1,000--1,500 µmol/mol in PFAL                       Cylinder + solenoid + NDIR CO2 sensor closed-loop dosing to zone-specific setpoint; interlocked with ventilation
  Airflow/HVAC                       Boundary-layer gas exchange, prevents tip-burn (Sec. 1.4)         CFD-optimized multi-inlet horizontal ducting, 0.2--0.5 m/s at canopy           Perforated duct or oscillating fan array per zone; Ooty ambient economizer-assisted cooling for saffron zone
  Insulation/envelope                Minimizes HVAC load, enables independent zone climates            PIR-core sandwich panel modular envelope (Sec. 2.2)                            80--100mm PIR sandwich panels, thermally broken joints, vapor barrier
  Irrigation/fertigation             Consistent nutrient delivery; largest labor-saving opportunity    PLC+HMI closed-loop pH/EC dosing (Sec. 1.6)                                    Peristaltic dosing pumps + inline pH/EC probes + microcontroller/PLC, target pH 5.6--6.2
  Hydroponics/substrate              Root-zone environment for microgreens & saffron corms             NFT/DWC for greens; soilless substrate for corms                               NFT trays (microgreens), coco/perlite substrate beds (saffron corms), emersed propagation trays (aquatics)
  Sensors/data logging               Enables reproducible R&D and process control (Sec. 1.7)           Multi-sensor fusion (env + imaging) with AI analytics                          Networked T/RH/CO2/pH/EC/PAR sensors logging to local + cloud DB; per-zone dashboards
  Imaging/phenotyping                Non-destructive growth tracking for product validation            RGB/multispectral time-lapse phenotyping rigs                                  Fixed RGB cameras per zone on a timer; optional low-cost NDVI camera for R&D crops
  Automation/IoT/controllers         Reduces labor, enables remote monitoring, is a sellable feature   Full SCADA/edge-AI control                                                     ESP32/PLC-based zone controllers + cloud dashboard (local-first, cloud-optional for reliability)
  Power systems                      Continuity of climate/lighting control is safety-critical         UPS + generator + solar hybrid                                                 UPS for controllers/sensors (data integrity), mains + manual generator backup for lighting/HVAC
  Safety systems                     Electrical/water/gas safety in a small enclosed room              Interlocked E-stops, leak/smoke detection, GFCI                                RCD/GFCI on all circuits, water-leak sensors under fertigation lines, CO2 leak alarm, smoke detector
  Modular mechanical design          Enables rack reconfiguration as crop mix evolves                  Tool-less modular aluminum extrusion racking                                   80/20-style aluminum extrusion racks, adjustable tier height, wheeled sub-modules per zone
  ---------------------------------- ----------------------------------------------------------------- ------------------------------------------------------------------------------ ------------------------------------------------------------------------------------------------------------------

2.5 Automation Tier and Controls Philosophy

Given the review findings in Section 1.7 that AI/ML-based autonomous
control still faces data and calibration constraints at small scale, the
recommended architecture is a local-first, rules-plus-feedback-loop
control system (setpoint + PID/fuzzy logic per zone) with cloud logging
and remote monitoring layered on top --- not a fully autonomous AI
controller. This is both the scientifically defensible choice for a room
this size (avoids over-claiming AI capability the data volume cannot
support) and the more reliable choice operationally, since local control
continues functioning during any internet or cloud outage. The same
architecture doubles as the R&D data-logging backbone: every zone\'s
environmental history and yield outcome should be timestamped and stored
so that the facility functions as a genuine product-validation
instrument, not just a production room.

2.6 Safety Systems

A small sealed CEA room concentrates several hazards that a larger
facility distributes across more space: electrical (lighting/pump
circuits near water), water (fertigation leaks, humidity systems), and
gas (CO2 enrichment in an occupied, low-air-exchange room). Recommended
baseline: RCD/GFCI protection on every wet-area circuit, under-tray
water-leak sensors wired to an automatic solenoid shutoff, an NDIR CO2
sensor with audible/visual alarm and interlocked ventilation override
(critical because CO2 concentrations in the 1,000--1,500 µmol/mol range
used for enrichment are asphyxiation-relevant if the room were ever
sealed with a person inside for extended periods at higher
concentrations), and a standard smoke/heat detector given the electrical
and lighting load density.

2.7 Cost-Performance Positioning

The subsystem choices above are deliberately calibrated to the
company\'s stated market position: above hobbyist/DIY grow-tent
equipment (fixed lights, no automation, no logging) but below full
industrial PFAL capital intensity (multi-ion AI dosing, robotic
handling, SCADA-grade redundancy). This positions the resulting product
line --- and the R&D room itself as a demonstrator --- as credibly
research-backed and engineering-led without requiring capital
expenditure or technical complexity that outstrips what a
first-generation Indian manufacturer, outsourcing fabrication locally,
can reliably build, service, and support. Section 2.4\'s \'Recommended
Mid-Tech Solution\' column is effectively a first-pass specification for
the Phase 1/Phase 2 product catalog to be developed in a later phase of
this report.

References

Anuar, N., Taha, R. M., Abdullah, S., Nazira, M., & Abdumutalovna, M. S.
(2024). Temperature effects on corm germination and flowering of Crocus
sativus L. (saffron). Journal of Animal and Plant Sciences, 34(1),
130--137.

Ahrazem, O., et al. (2015). Effects of ambient temperature on flower
initiation and flowering in saffron (Crocus sativus L.). Scientia
Horticulturae.

Molina, R. V., et al. (2005). Temperature effects on flower formation in
saffron (Crocus sativus L.). Scientia Horticulturae.

Kozai, T. (2013). Resource use efficiency of closed plant production
system with artificial light: Concept, estimation and application to
plant factory. Proceedings of the Japan Academy, Series B, 89(10),
447--461.

Li, K., et al. (2021). Carbon dioxide enrichment promoted the growth,
yield, and light-use efficiency of lettuce in a plant factory with
artificial lighting. Agronomy Journal, 113(6).

Wang, A., Lv, J., Wang, J., & Shi, K. (2022). CO2 enrichment in
greenhouse production: Towards a sustainable approach. Frontiers in
Plant Science, 13, 1029901.

Dannehl, D., Kläring, H.-P., & Schmidt, U. (2021). Light-mediated
reduction in photosynthesis in closed greenhouses can be compensated for
by CO2 enrichment in tomato production. Plants, 10(12), 2808.

Wu, B. (2016). A CFD study on improving air flow uniformity in indoor
plant factory system. Biosystems Engineering, 147, 193--205.

Kitaya, Y. (2005). Effects of air current on transpiration and net
photosynthetic rates of plants in a closed plant production system.
Environment Control in Biology.

(2022). Analysis of climate uniformity in indoor plant factory system
with computational fluid dynamics (CFD). Computers and Electronics in
Agriculture.

Van Delden, S. H., et al. / Benchmarking energy efficiency in vertical
farming: Status and prospects. (2024). Sustainable Production and
Consumption.

Zhang, Y., et al. (2024). Energy consumption of plant factory with
artificial light: Challenges and opportunities. Renewable and
Sustainable Energy Reviews.

Sheibani, F., & Mitchell, C. A. (2023). Close-canopy lighting, an
effective energy-saving strategy for overhead sole-source LED lighting
in indoor farming. Frontiers in Plant Science, 14, 1215919.

Colantoni, A., et al. (2024). Lighting strategies in vertical urban
farming for enhancement of plant productivity and energy consumption.
Journal of Cleaner Production.

Kaya, C. (2025). Intelligent environmental control in plant factories:
Integrating sensors, automation, and AI for optimal crop production.
Food and Energy Security, 14(1), e70026.

(2026). Smart greenhouses in the era of IoT and AI: A comprehensive
review of AI applications, spectral sensing, multimodal data fusion, and
intelligent systems. Agriculture, 16(7), 761.

(2025). Multi-sensor monitoring, intelligent control, and data
processing for smart greenhouse environment management. PMC review
article.

Bethge, H., Winkelmann, T., Lüdeke, P., & Rath, T. (2023). Low-cost and
automated phenotyping system \"Phenomenon\" for multi-sensor in situ
monitoring in plant in vitro culture. Plant Methods, 19, 42.

Dobón-Suárez, A., et al. (2024). Can LED lighting be a sustainable
solution for producing nutritionally valuable microgreens? Foods /
Horticulturae, 10(3), 249.

(2026). Effects of LED lighting on the nutritional properties and
microbial safety of microgreens. Frontiers in Nutrition, 13, 1869208.

Brazaitytė, A., et al. (2021). Effects of different light spectra on
final biomass production and nutritional quality of two microgreens.
Plants, 10(8), 1584.

Kyriacou, M. C., et al. (2022). Continuous LED lighting enhances yield
and nutritional value of four genotypes of Brassicaceae microgreens.
Plants, 11(2), 217.

(2026). Blue-enriched LED light modulates biochemical and proteomic
traits without affecting yield in indoor-grown cress microgreens.
Frontiers in Plant Science, 17, 1814329.

(2026). Manipulation of light intensity and spectrum for driving growth
and metabolism to improve nutritional and nutraceutical quality of
microgreens. Plant Growth Regulation.

Vanegas-Ayala, S. C., Barón-Velandia, J., & Leal-Lara, D. D. (2022). A
systematic review of greenhouse humidity prediction and control models
using fuzzy inference systems. Advances in Human-Computer Interaction,
2022, 8483003.

(2025). Forecasting the vapor pressure deficit in vertical farming
facilities aiming to provide optimal indoor conditions. Journal of
Agricultural Engineering, 56(3).

(2025). A comprehensive review of advances in sensing and monitoring
technologies for precision hydroponic cultivation. Computers and
Electronics in Agriculture.

(2025). Automated hydroponic nutrient dosing system: A scoping review of
pH and electrical conductivity dosing frameworks. Preprint/ResearchGate.

Ruiz-Garcia, L., et al. (2018). Smart system for bicarbonate control in
irrigation for hydroponic precision farming. Sensors, 18(5).

(2025). Towards sustainable vertical farming: A systematic review of
energy return on investment efficiency and optimization strategies.
Sustainability, 17(18), 8142.

(2025). Effect of Trichoderma harzianum and nano-TiO2 treatments on
saffron (Crocus sativus L.) cormlet production under high-temperature
stress. Scientia Horticulturae / ScienceDirect.

Buce Plant / South Scape Aquatica / MB Store / Horizon Aquatics
(2023--2026). Industry guides on emersed vs. submersed cultivation of
aquarium plants for commercial nursery propagation. \[Trade/industry
sources; not peer-reviewed --- cited for commercial practice only.\]

Climate data compiled from Climate-Data.org, Weather Atlas, Weather
Spark, and OotyMade.com for Udhagamandalam (Ooty), Tamil Nadu (2,240 m
altitude, Köppen Cwb/Cfb subtropical highland climate).
