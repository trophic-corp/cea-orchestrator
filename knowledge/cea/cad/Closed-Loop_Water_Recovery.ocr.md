## Page 01
```
RK-A-WRS REV 3

Closed-Loop
Water Recovery

Both reservoirs move to the terrace, one floor above the room. That single decision

05 SEP 2026 TERRACE RESERVOIR, GRAVITY FEED

deletes the supply pump, lets every bed flood by gravity, and turns the recovery side
into one trough, one pump and one rising main. It also introduces three risks that did

not exist before, and they are the most important part of this document.

CIRCULATED

1,914 L/day

PUMPS IN SYSTEM
iduty+1
standby

MAKE-UP

235 L/day

SKID COST

79k / 153k

O01 - THE TERRACE DECISION

RECOVERY RATIO

87.7%

DISCHARGE CUT

554 m?/yr

What moving the tanks upstairs actually buys

Rev 2 had a floor-standing 500 L reservoir, a supply pump, a catch trough, a transfer

pump, a recovery reservoir and a return pump. Putting the reservoir on the terrace

collapses most of that.

ELEMENT

Supply pump
P-01

Recovery
reservoir TK-
03

Return pump
P-03

REV 2 — PLANT IN THE ROOM

0.55 kW, 30 L/min @ 18 m, runs
every flood

300 L at floor level, plus a return

pump

0.25 kW

REV 3 — RESERVOIR ON THE TERRACE

DELETED Gravity does it. 2.66 m of head at the top
tier against 0.33 m of loss

DELETED The gate is now decided in the trough,

before anything is pumped

DELETED
```

## Page 02
```
Iranster pump U.3/ KW, ZOL/MIN@ IZM Uprated To U./9 KW, Z9 L/min @ ZUM — ITNoOw IITts

P-O2 to the terrace as well as pushing through the filters

Room floor 5.73 m? bay 1.02 m? service corner — one rack instead of three
area for plant

Racks in the 9 11: grow area 29.0 m?: floor multiplier 1.30*

room

Fill One tier at a time, pump-limited Up to four tiers at once — gravity has the head to
concurrency spare. Room cycle 88 min > 22 min

Carrying To the room Not to the roof — dosing injects into the rising main
fertiliser salts at floor level

The dosing point is the detail that makes this work. The obvious objection to a roof tank is that
someone has to carry 25 kg bags of A and B salts up a ladder three times a week. They don't: the
dosing skid stays at floor level and injects into the rising main, so every litre of recovered water is
dosed on its way up and mixed by its own inflow into the tank. EC in TK-01 is the feedback

signal. The only thing that ever goes up there is water.

02 - GRAVITY FEED
Does 3.75 m of terrace actually flood a bed at 1.5 m?

Comfortably, and with enough spare head to flood all four tiers at once. The arithmetic

matters because it is what deletes the supply pump.

terrace slab +3750 mm above room floor

tank on a 500 mm plinth > outlet at +4250 mm, worst case is a nearly-empty tank
top tier fill nozzle +1590 mm

available static head = 4250 - 1590 = 2.66 m

Losses at the design fill rate of 7.2 L/min per tier (Hazen-Williams, C = 150):
DN32 down-main, 14 m 0.028 m

DN25 riser, 1.5 m 0.012 m

DN20 tier drop, 1.0 m 0.023 m

DN20 fill solenoid, Kv 4 0.119 m « the dominant loss

fittings 0.150 m

total 0.33 m + margin 2.33 m - passes by a factor of eight
```

## Page 03
```
with all four tiers of a rack filling together, 28.8 L/min in the down-main:
total 0.67 m + margin 1.99 m - still passes

head runs out at roughly 15 L/min per tier, twice the design rate

Two consequences follow. First, there is no supply pump anywhere in the system — the only
pump is the one lifting recovered water back up. Second, the sequential-filling rule that existed
to protect a 7.2 L/min pump is no longer needed on the supply side. A rack can flood all four
tiers at once, which takes the whole-room fill window for eleven racks from 88 minutes to
about 22.

Sequential draining is still mandatory. Nothing about the terrace changes the DN50 drain
stack or the size of the catch trough. Draining stays one tier at a time, and — new in this revision
— one rack per recovery batch, because the gate is decided on a settled batch and the trough
holds 79 L against a 46 L rack drain.

03 - PROCESS

The loop, end to end

TERRACE yee

 

TK-01
MAIN RESERVOIR 500 L
insulated + shaded - opaque
on 500 plinth over a beam Line

TK-04
break tank 100 L >

type AB air gap

 

 

 

 

=o AToOL te AT 02 pit TE“ oT

 

 

MV-01

master valve - f##ls CLOSED

°
Eo
©
4
a
e

GROWING ROOM + FFL O a
n SS cuheateakebhertonetiegn point
st

RACK x 11 - 4 TIERS EACH - 44 BEDS

 

bed 1176 x 560 - flood 22 mm - 14.5 L + fill NG

DN20 RISING MAIN -

moulded 30 mm overflow collar + DN32

tray bulkhead DN40 + Lift-out strainer

drain solenoid NO - DNS5@ common header

 

 

 

 

| DOSING ] nts
```

## Page 04
```
A +B + acid
at floor level

 

 

 

 

F-02 F-01
UV-01

|< 20" CART <_j- BAG

40 W - 108 mJ/cm 20 pm 160 pm

ATP O3
P-02 A/B
UIT-01 intensity - FS-01 flow
de-energi PDI-01 across F-@1 + F-@2 + inhibit at 1.0 bar
to drain

FIG 1 — REV 3 PROCESS FLOW. BLUE IS GRAVITY, TEAL IS PUMPED RECOVERY, AMBER IS DOSING, RED
IS THE WASTE PATH AND THE FAIL-SAFE POSITIONS. ONE PUMP, ONE RISING MAIN, ONE CROSSING OF
THE CLEAN/UNVERIFIED BOUNDARY AT FV-01.

 

 

 

 

 

 

 

 

 

4

04 - NEW RISKS

Three things that did not exist when the tanks were indoors

This is the part of the terrace decision that has to be engineered rather than assumed.
All three are manageable; none is optional.

Risk 1 — solution temperature

A tank on a terrace is a solar collector. In Coimbatore, an unshaded HDPE tank routinely reaches
40 °C by mid-afternoon in April and May. Nutrient solution above 26 °C loses dissolved oxygen
and becomes hospitable to Pythium — itis gate G3 in the reuse logic, and here it would be

failing on the supply side, where there is no gate at all.

dissolved 02 at 20 °C 9.1 mg/L

dissolved 0, at 30 °C 7.6 mg/L — a 16% Loss before any root has used a molecule
dissolved 0. at 40 °C 6.4 mg/L — 30% Loss, and Pythium zoospore activity peaks in
this band

Minimum — Ooty Required — Coimbatore

Opaque tank under a shade structure, 25 mm Everything above, plus temperature logged
PUF wrap with white UPVC cladding, and continuously with an alarm at 26 °C, and
fill/draw scheduled so the tank turns over budget for a 72 TR inline water chiller. Assume
overnight. Ooty's ambient does the rest. you will need it; be pleased if the data says

Sufficient for the first six months. otherwise.
```

## Page 05
```
Risk 2 — a500L tank with a gravity path into the room

A burst hose or a failed union downstairs, with the tank full, empties 500 litres onto the growing
room floor. Nothing in the Rev 2 architecture could do that, because the reservoir sat on the
same floor as the leak.

e MV-01, motorised master valve at the tank outlet, spring-return closed. Energised only
during a flood window. Everything downstream of it is dry between floods.

e Vacuum breaker at the down-main high point, so a downstream failure cannot siphon the
tank.

¢ Room flood sensor de-energises MV-01 directly, hard-wired, not through the bus.

e Flow totaliser on the down-main. A flood window that delivers more than 1.3 x the expected
volume closes MV-01 and alarms — this catches a slow leak that a flood sensor at one floor
point would miss.

Risk 3 — terrace slab loading

TK-01 filled 500 L + tank 25 kg + plinth = 560 kg

TK-04 filled 100 L + tank = 110 kg

on a 1.0 m? footprint = 5.5 KN/m? — an accessible terrace is typically designed for
1.5-2.0

spread over a 2.0 x 1.5 m platform bearing on beam Lines:
670 kg / 3.0 m? = 2.2 KN/m? — at the Limit, and only acceptable over beams, never
mid-span

This needs a structural engineer's sign-off before the tank is ordered, not after. The load is
concentrated, permanent and sits on a slab that was probably designed for foot traffic. The
platform must bear on the beam or column lines, and the check has to be done against the
building's actual drawings. It is the one item in this document that cannot be resolved by
choosing better equipment.

05 - WATER BALANCE

Eleven racks, 44 beds, three cycles a day
```

## Page 06
```
STREAM

Gross flood delivered

Evapotranspiration

Water leaving with

harvest

Substrate charge on re-
sowing

Returned to trough

Blowdown for salt control

Make-up required

recovery ratio = 1 - 235 / 1,914
annual discharge, closed Loop 48.
annual discharge, drain-to-waste

L / CYCLE

638

14.5

6.3

8.8

608

48.7

78.3

reduction = 554 m°/yr - 92%

L / DAY M> / YR

1,914 6352
43.5 14.4
19.0 6.3
26.4 8.7
1,825 602

146 48.2

235 77.6

87.7%

2 ms
632 m?

BASIS

44 beds x 14.5 L (1176 x 560 x 22

mm)

1.5 Lm*d™ at DLI 10.4 x 29.0 m?

canopy

4.8 kg fresh per bed, 90% water,
10-day cycle

6 L per bed per 10-day cycle

gross — consumption

8% of returned volume, or as Na*
demands

consumption 88.9 + blowdown
146

One rack per recovery batch. The trough holds 79 L of working volume against a full rack drain

of 46 L, and the reuse gate is decided on a settled, mixed batch rather than a moving stream.

That sets the cadence: drain one rack (2.7 min), settle (80 s), test (8 min), transfer (2 min) —

about 8 minutes per rack, 88 minutes for eleven, three times a day. It is the drain side, not the fill
side, that now sets the room's clock.

06 - EQUIPMENT

What is left, after the terrace deleted three items

TAG ITEM

TK- Main nutrient

LOCATION

Terrace

DUTY / SPECIFICATION

500 L opaque HDPE. 25 mm

WHY

Opaaue against alaae,
```

## Page 07
```
01

TK-
04

MV -
01

VB-
01

TK-
02

$-01

P-02
A/B

F-@1

F-02

UV-
01

FV-
01

reservoir

Make-up
break tank

Master supply
valve

Vacuum
breaker

Catch trough

Basket strainer

Transfer / lift
pump, duty +
standby

Bag filter

Cartridge filter

UV-C
disinfection

Diverter valve

Terrace

Terrace,
tank outlet

Down-main
high point

Room, NE
service

corner

In TK-O2

Room,
service

corner

Skid frame
over TK-O2

Skid frame

Skid frame

Skid frame,
before the
rising main

PUF + white cladding, shaded,
ona2.0 x 1.5 m spread platform

100 L, type AB air gap, float
valve, 5 um + carbon pre-filter

DN32 motorised, spring-return
closed

DN32 anti-siphon

1400 x 650 x 200 mm fabricated
HDPE, inlet at 87 mm, 79 L
working, bunded lip

200 um SS316, lift-out

0.75 kW, 25 L/min @ 20 m, self-
priming, EPDM seals, IP55, auto-
changeover

100 um, size-1 bag, 3 spares

20-inch wound PP, 20 um, 5

spares

40 W amalgam, 316L, quartz
sleeve, intensity sensor:1.5 m3/h
at 100 mJ/cm?

DN25 motorised 3-way, spring-
return to waste

insulated against the
sun, spread against the
slab

Backflow prevention
belongs at the highest
point in the system

The only thing standing
between 500 L and the

room

A downstream failure
must not siphon the tank

Shallow because tier 1
discharges at 170 mm; a
standard drum cannot
be gravity-fed

Root hairs and coir fines,
before they reach an
impeller

One pump now does
filtration, disinfection
and the 4.76 m lift.
Standby because there is
no second path

Bulk solids; cheaper per
gram of dirt thana
cartridge

20-inch, not 10-inch — a
single 10" element is
rated 11-15 L/min and
would throttle P-O2

100 mJ/cm? is the
horticultural design dose
for oomycete and
bacterial control

Unverified water never
enters the rising main
without power and a
```

## Page 08
```
DOS -

01

Dosing skid

P-02 duty check
static Lift, trough water 87 mm > tank inlet 4850 mm 4.76 m
DN20 rising main, 14 m at 25 L/min 1.13 m

F-01 + F-02 clean 3.70 m

UV-01 2.80 m
fittings and the diverter 0.60 m

clean duty 12.4 m -

07

The reuse decision moves upstream

Room, floor
level

INSTRUMENTS AND THE GATE

A+B +acid peristaltic, injecting
into the rising main

at the 1.0 bar filter-change Limit 18.9 m
specify 25 L/min at 20 m — 0.75 kW, not the 0.37 KW of Rev 2

passed gate

Nobody carries fertiliser
salts to a roof

In Rev 2 the water was filtered, disinfected and stored, and only then tested. In Rev 3 it

is tested first, in the trough, and only water that passes is ever pumped. That saves a

tank, appump and the energy of lifting water you are about to throw away.

GATE

G1

G2

G3

G4

G5

G6

CONDITION

Electrical
conductivity

pH

Temperature

Filter differential

pressure

Disinfection proven

Leak detectors

clear

THRESHOLD

< 15 setpoint — < 180
mS/cm at a 1.20 setpoint

45-75

< 26°C

< 1.0 bar across F-O1 + F-

02

UIT-O1 = 70% and lamp
on and FS-01 made

No leak puck active this

cycle

MEASURED AT

AT-O38, in the
trough

AT-04, in the
trough

TE-O2, in the

trough

PDI-O1

UV-01

LD-O1...11

ON FAILURE

Trough dump valve to

waste

Dump to waste

Hold 60 min, re-test, then
dump

Inhibit P-O2, alarm,
change elements

FV-O1 to waste mid-
transfer

Dump to waste; isolate
that rack
```

## Page 09
```
G7

G8

Operator Not set Controller Dump to waste until

contamination hold cleared by anamed
operator

TK-O1 has room LT-O2 < 90% Terrace Hold in trough until level
falls

Sequence for one recovery batch

01

02

03

04

05

06

07

08

09

One rack drains. Four tiers in sequence into the trough — 46 L. LSH-04 confirms each header
is empty before the next tier.

Settle 30 s. Solids to the strainer basket, air out of the sample.

Test 3 min. AT-03, AT-04 and TE-O2 read as 60-second rolling means on the mixed batch, not
spot values on a moving stream.

Evaluate G1, G2, G3, G6, G7, G8. Fail on any — the trough dump valve opens to waste and the
event is logged with the failing gate named. Nothing is pumped.

UV pre-strike. On a pass, UV-01 energises and holds 60 s of amalgam warm-up before the

pump starts.

Transfer. P-O2 runs; FS-O1 must make within 10 s. G4 and G5 are watched continuously —
either failing throws FV-O1 to waste mid-transfer.

Dose on the way up. DOS-01 injects A, B and acid into the rising main, proportional to the
trough EC deficit against setpoint.

Stop. P-02 stops on LS-01 low or a 4-minute run limit — the run limit is what catches a stuck
float.

Blowdown. Daily at 02:00, the first batch is dumped regardless of gate result. This is what

stops sodium accumulating; it is not an error condition.

The supply side has no gate. Water in TK-O1 has already passed once, but it sits on a roof

between uses, warming and concentrating by evaporation. TK-O1 therefore carries its own EC,

pH and temperature block, read before every flood window, with MV-01 inhibited if temperature
```

## Page 10
```
exceeds 28 °C. That is a supply-side interlock, not a reuse gate, and it is the direct consequence
of putting the tank in the sun.

08 + FAIL-SAFES

De-energised states
DEVICE DE-ENERGISED REASON
M\V-01 terrace CLOSED 500 L above the room has exactly one thing holding it back,
master valve and it must not be electricity
Rack fill solenoids CLOSED No bed fills unattended
Rack drain solenoids OPEN Beds self-drain; no standing water on a dead rack
FV-O1 diverter TO WASTE Unverified water never enters the rising main
Trough dump valve CLOSED A power cut must not silently discard a good batch
Make-up float valve MECHANICAL Float, not solenoid — TK-04 keeps working through a power cut

P-02, UV-01, DOS-01 OFF —

Every vessel keeps a gravity overflow one nominal size larger than its largest inlet, terminating
with a visible air gap: TK-O1 and TK-04 overflow to the terrace rainwater outlet, TK-O2 to the
room's waste. An overflow discharging into a closed pipe is not an overflow — it isa second way
to pressurise the tank.

Recirculation still spreads root disease, and the terrace does not change that. A shared loop
connects all 44 beds hydraulically; Pythium and Fusarium zoospores travel in solution and one
infected bed can inoculate the room in two cycles. UV-C at = 100 mJ/cm?, 20 um pre-filtration,
and the G7 operator hold that drops the room to drain-to-waste in one button press are the
three controls, and none is optional at production scale. UV-C also degrades Fe-EDTA —
specify Fe-DTPA or Fe-EDDHA and verify iron by lab test monthly, or you will diagnose a pH
problem that is actually a chelate problem, two weeks late.

 
```

## Page 11
```
09 + COST

Two build levels

ITEM QTY RATE & R1 & R2 ADD 2

TK-01500L 1 14,200 14,200 -
opaque HDPE

+ PUF

insulation +

cladding

Terrace 1 11,500 11,500 -
platform, Gl,
spread over

beam lines

~ . a ao 4aAn a an
```

