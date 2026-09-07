## Page 01
```
RK-A-SYS + REV 2 + 04 SEP 2026 + PHASE 1 BASELINE + DRAINAGE SUPERSEDED BY
RK-A-WRS REV 2

Rack A
Systems Specification

The engineering baseline for one fully specified rack: water, air, light, sensing and
power, with the interfaces defined so a room of these scales without redesign. This
document is the OC reference against which the built prototype is validated.

WATER AIR POWER RACK SENSORS
14.5L / tier 178 m?/h / tier 316 W / rack 2

BUILD COST OPEN ITEMS

51,256 2-both tests

Partly superseded. The drain-to-waste strategy in this document has been replaced by a
closed-loop water recovery system — see Closed-Loop Water Recovery (RK-A-WRS Rev 2).
The rack-side drainage hardware is unchanged; what changed is its destination, the fail-safe
polarity of the diverter, and four control behaviours. Nineteen corrections from the integrated-
model validation are recorded in the Engineering Validation Record (RK-A-OC Rev 1); the
integrated rack envelope is now 1456 x 690 x 1960 mm at 125.4 kg dry.

 

00 - STATUS

What this document changes

Specifying the subsystems properly rather than estimating them moved the cost along
way. That is the headline and it comes first.

LINE PLACEHOLDER ENGINEERED WHY
Irrigation + 3,500 8,950 Per-tier fill and drain valving, bulkheads, overflow
drainage standpipes, strainers, isolation

Electrical + 3,200 9,580 IP65 enclosure, isolator, RCBO, 24 V PSU, rack
```

## Page 02
```
control

Ventilation

LED mounting

Rack sensors

Reflective

panels

Grow total

%2,6354

1,905

332,965

=5,520

=5, 460

%1,650

removed

351,256

controller, segregated cabling

Was never costed. Four EC fans and four perforated

plenums

Hangers, saddles, IP65 connectors — rails alone were
costed before

Was never costed

Per your instruction. See the caveat below

+55% — see §11 for the Rev 2 deltas

~

 

01 - DESIGN RULES

genuinely redundant to the airflow function.

Correction — | double-counted, and you were right to push back. Rev 1 carried two separate
parts doing one job: a folded GI plenum with a perforated face inside the 5,520 ventilation line,
anda 600 PP "plenum face”. The GI plenum already isthe perforated duct that distributes air
along the full 1176 mm. The PP panel was the old reflective rear panel repurposed, and itis

So the PP rear panel is now optional, as you asked, and the airflow system is unaffected by
removing it. 3600 comes out of the base rack. What the optional panel buys is tier enclosure —
stopping air escaping sideways — plus the light recovery if specified in matte white.

What the optional panels still buy. Distribution along the tier is handled by the GI plenum and
is not at risk. What is at risk without side and rear enclosure panels is containment. some supply
air escapes sideways instead of crossing the canopy, so velocity uniformity will be worse than
the ~10% coefficient of variation the CFD literature reports for a fully ducted tier. Measurable,
not fatal — check canopy velocity at five points per tier at commissioning, and if the coefficient
of variation exceeds 15%, fit the panels. The grid holes are already there.

Five rules everything else follows from

Water never shares a route with power

Every fill point has a physical air gap

>
```

## Page 03
```
All water is in the rear service zone and below
bed level. All electrical is above canopy or in the
top-rear enclosure. Minimum 236 mm vertical
separation, and no cable crosses a wet zone

without a drip loop.

Flood depth is set mechanically

An overflow standpipe in each tray sets the
level. A stuck-open fill valve overflows to drain
rather than over the rim. The timer sets
duration; the standpipe sets depth.

The grid is the only mounting interface

Every bracket, clip, sensor, plenum and rail
bolts to the 50 mm hole grid. Nothing is drilled,

welded or bonded to the frame in the field.

IRRIGATION

Water into the rack

RACK PLUMBING SCHEMATIC

one rack - 4 tiers - fill from above with air gap - gravity drain

ROOM - SHARED
ISOLATION

DN25-32pHAIN
RESERVOIR _HKO—
DN20 RISER

Trays are filled from a nozzle above the rim, not

through a bottom port. Back-siphonage from a

tray into the supply is then impossible by

geometry rather than by a valve that can fail.

Any rack can be isolated alone

One ball valve and one union on water, one

isolator and one RCBO on power, one

addressable node on the bus. A rack comes out

of service without stopping the room.

RACK BOUNDARY — everything inside ships with the rack

DN40 DRAIN HEADER

 

 

PUMP BV STR UN OF

pH - EC + TEMP + LEVEL

 

dosing - filtration

 

 

LEGEND

 

 

 

 

 

 

 

 

 

© Fitt sotencid, 12 V O
® drain solenoid, DN40

a overflow standpipe

X ball valve / union

— supply / drain line

 

AIR GAP PJoverFLow

TIER 4 FLOOD TRAY 14.5 L | /K—— (0)
in

TIER 3 | t+—— (@)}-—
nh.

TIER 2 1 /—— (0)
fh.

TIER 1 | t+——(@)}—

 

 

 

 
```

## Page 04
```
wy

LEAK -SENSOR ROOM DRAIN DN63

RK-A-P01 + RACK PLUMBING SCHEMATIC - SUPPLY DN20, DROPS DN16, DRAIN HEADER DN40, ROOM
DRAIN DN63
— >

Connection to the rack, and where

POINT

Supply
inlet

Inline

strainer

Riser

Tier drop

Tray drain

Tray

overflow

Drain

header

Discharge

LOCATION ON RACK

Left rear upright, 250 mm above

floor, rear service zone

Immediately after the inlet valve,
serviceable from the aisle

Left rear upright, clipped to the 50
mm grid

Rear of each tier, terminating

above the tray rim

Right end of tray floor, low point

Adjacent bulkhead, standpipe

crest at 30 mm

Right rear upright, vertical

Base of the right rear upright, 150
mm above floor

SIZE

DN20

DN20, 120

mesh

DN20 PE

DN16 PE

DN4O

DN40 boss,
DN32 pipe

DN40 uPVC

DN40

FITTING

PP ball valve + union, EPDM seat —

the rack disconnects at the union

PP body, clear bowl, EPDM O-ring

Push-fit, P-clips at 400 mm

Push-fit elbow, nozzle clipped over

the rim

PP tank connector, EPDM washer on
the wet face, removable mesh
strainer

PP tank connector + loose standpipe,
tethered

Clamp couplings so the stack lifts out

without dismantling the rack

Air gap over the room drain — never

a direct connection

( EE >

How water actually reaches a tray

The tier drop terminates in a nozzle clipped over the tray rim, discharging downward into open

air. There is no bottom fill port, no seal below the water line on the supply side, and a physical air

gap at every one of the sixteen fill points. Back-siphonage of nutrient solution into the supply

main is prevented by geometry rather than by a check valve that can stick. It also deletes four

seals per rack that would otherwise be candidates for leaking.
```

## Page 05
```
Why the flood depth is mechanical

Each tray carries an overflow standpipe with its crest at 30 mm — above the 22 mm working
flood, below the 47 mm rim. The fill solenoid runs on a timer, but if it sticks open the tray simply
overflows down the standpipe into the drain header. The timer sets duration; the standpipe sets
depth. This is the single most important safety feature in the water system and it has no moving
parts.

Standpipe sizing caught a near miss. A DN25 standpipe passes 7.2 L/min at 8 mm of head over
the crest — exactly the fill rate, so it would run at the point of flooding with no margin. DN32
minimum, which passes 11.9 L/min, a 1.65* margin. Specify DN32 and do not let a supplier
substitute DN25 because it fits the same boss.

Sequential filling, and why it matters at room scale

STRATEGY PUMP DUTY SUPPLY SIZE CYCLE VERDICT
Sequential — one tier at a time 7.2 L/min  DN20 branch 24 min Adopted
Simultaneous — all four tiers 29.0 L/min DN25-32 branch 13.5 min = Rejected

a >

Sequential filling costs eleven minutes of cycle time — irrelevant at one to three cycles a day —
and buys a quarter of the pump duty and a smaller pipe. The real payoff appears at room scale:
because the room controller fires one tier at a time across all racks, the pump and the main
never grow with rack count. Twenty racks need the same 7.2 L/min pump as one. That is the

property that makes this scale cleanly.

Seals and wetted materials

APPLICATION MATERIAL REASONING
All gaskets, O-rings, EPDM, food grade Excellent with water at pH 5.6-6.2, and — the deciding
valve diaphragms (WRAS/FDA) factor — compatible with both hypochlorite and peracetic

acid, so the sanitiser decision cannot strand it

Not to be used NBR / Buna-N Poor ozone and UV resistance; degrades in a lit, humid

room

Not required Viton / FKM Chemically excellent, several times the cost, no benefit at

thaen ahamictrine
```

## Page 06
```
LIITOS VLLITITMOLUITS

Valve and fitting PP or PVC-U, food No metal in the wetted path — avoids both corrosion and
bodies grade metal ion pickup into the nutrient
Supply pipe PE (LDPE) Standard Indian drip-irrigation ecosystem, push-fit,
DN20/DN16 cheap, food safe, tolerant of movement
Drain pipe uPVC DN40 to IS Rigid, solvent-welded, stocked everywhere
4985
Tank connectors PP body, EPDM Gasket on the wet side is standard practice — the water
washer on the wet pressure seats the seal rather than trying to lift it
face
Fasteners in the wet $S$304 with nylon See the galvanic note in §10 — this is a real finding, not
zone isolating washer boilerplate

(ED >

Routing so it does not get in the way
e All water lives in the rear 90 mm service zone. The riser and header run on the rear faces of
the left and right uprights, clipped to the grid, outside the tray envelope.

e Tier drops cross the rear plenum face through grommeted holes at a fixed position, then
terminate over the tray rim. They do not cross the canopy, the LED plane, or the front access
opening.

e The drain header is on the opposite upright from the supply riser, so a leak at either is
unambiguous.

e Nothing water-carrying passes above bed level except the tier drop, which is above the rim

and downstream of the air gap.

e Trays lift out upward with no plumbing attached to them except the two bulkheads, which

disconnect at the clamp coupling on the header.

Multi-rack: how it scales

QUESTION ANSWER

Common irrigation Yes. One reservoir, one pump, one dosing set per room. DN32 supply main run along
system? the rear wall, DN63 drain main below it at 1:100 fall.

Independent valve Yes — mandatory. A manual ball valve plus a union at every rack inlet. This is what

per rack? allows a rack to be worked on while the room runs.

Indanandant flaw Mat naadad and warth aunidina Elavwsic cat nar tiar hw tha enlannidA anan tima
```

## Page 07
```
IIMOGVONMUOGIIL Tu

control per rack?

Isolating one rack

for maintenance

What grows with
rack count

wwe pwcucuy, CALIWA VWWQZE ULE CAV WIE wy 1ivvvy iv owe vw uw wy LIIws OVIVCIIVIU vewi ruriiw,
which the controller already owns. A balancing valve per rack is one more thing to be
set wrong and drift. Add a per-rack flow meter on the Pro build for diagnostics, not

control.

Close the branch ball valve, break the union, lift the drain header clamp, open the
rack isolator, pull the RS485 drop. Four actions, no room shutdown.

Only the length of the two mains and the number of branches. Pump, reservoir and
dosing are sized by concurrent demand, which sequencing holds at one tier.

ee >

03 - DRAINAGE

Water out of the rack
PARAMETER VALUE BASIS
Method Gravity throughout the No pump above floor level. The only pump in
rack the system is on the supply side.
Drain rate, DN40 at 22 30.7 L/min 4.2x the fill rate

mm head

Drain rate at 30 mm

(overflow)

Tray empty time

Automated?

Tray fall to drain

Strainer

Discharge

35.9 L/min 5.0x fill
= 55 s Declining head
Yes — one DN40 drain Closed during fill and dwell, opened to drain

solenoid per tier

2-3 mm across the floor Formed into the tray so no puddle is left behind
Removable mesh over the Keeps media and roots out of the header
drain boss

Air gap at 150 mm above Never a direct connection to the room drain
floor

SD >

Why the overflow is a separate bulkhead rather than a concentric standpipe. The classic

ebb-and-flow tray uses one boss with an inner overflow tube inside an outer drain screen. That
```

## Page 08
```
fails here in the one case that matters: if the boss blocks with media, both the drain andthe

overflow are lost at the same instant. Two independent penetrations mean a blocked drain boss

still has a working overflow path. A blockage in the shared header defeats both, and that

residual case is covered by the leak sensor and the solenoid interlock, not by geometry — itis

listed in the FMEA.

04 - SENSORS

What is measured, and where

The governing principle is that a sensor belongs at the level where the quantity actually

varies. Putting a CO2 sensor on every rack measures the same room sixteen times and

costs sixteen times as much.

SENSOR LEVEL QTY

Air temperature RACK 1

+RH

Leak detection RACK 1

Air temp + RH, TIER 4

per tier

PPFD / PAR PORTABLE 1 per
site

Leaf TIER opt.

temperature

(true VPD)

pH WATER 1

EC/TDS WATER 1

Water WATER 1

temperature

EXACT LOCATION

Front-right upright, on the grid, in a vented
radiation shield, mid-depth of tier 3

Floor spot sensor at the base of the drain
header, right rear

Pro / R&D build only. Same mounting rule,
one per tier

Quantum sensor on a grid-mount jig. Five
points per tier at commissioning and after

any fixture height change

Pro only. |R thermometer on the grid,

aimed at canopy centre at 45°

Reservoir or inline on the pump return,

accessible for calibration without tools

Alongside pH, same manifold

In the reservoir, mid-depth

HEIGHT
1239 mm —
tier 3

canopy

5 mm above
floor

canopy of
each tier

canopy Level

500 mm above
canopy

reservoir

reservoir

reservoir
```

## Page 09
```
Reservoir level

Flow rate

Supply pressure

CO, (NDIR)

Room air temp +
RH

Smoke / heat

Room flood

WATER

WATER

WATER

ROOM

ROOM

ROOM

ROOM

1-2

1

Ultrasonic above the surface, or a float

switch pair for low and low-low

Inline on the supply main after the pump.
Per-rack flow is a Pro option for
diagnostics

After the pump, for dry-run and blockage
protection

Mid-room at canopy height, away from
doors and the HVAC discharge

One at HVAC supply, one at return — the
pair gives you the room's actual duty

Ceiling, per the room's fire design

Floor low point, near the drain

reservoir

plant room

plant room

1200 mm

supply /
return

ceiling

floor

ee >

 

Two sensors that people buy and should not. There is no such thing as a VPD sensor — VPD is

computed from air temperature, relative humidity and leaf temperature. In the standard build it

is calculated from the rack T/RH node with leaf assumed equal to air, which is adequate for

control; true VPD needs the IR leaf sensor and belongs only on the R&D rack. And there is no

per-tray water level sensor in this design at all: the overflow standpipe sets level mechanically,

which deletes sixteen sensors per rack and replaces them with a piece of pipe that cannot drift,

fail or need calibration.

Mounting rules — how sensors avoid interference

Minimum 150 mm from any LED fixture. Closer and the sensor reads fixture radiant heat,

not air, and will report a temperature that no plant experiences.

Never in the direct plenum jet. Mount at mid-depth of the tier, not against the perforated

face — asensor in the jet reads supply air, not canopy air.

Always in a vented radiation shield. A bare sensor under horticultural lighting over-reads by

a degree or more.

Above the tray rim, always. No sensor may sit where a flood or an overflow can reach it.

Out of the tray path. Nothing mounted where a tray is slid in and out — sensors go on the

front-right upright face, clear of the 1176 mm tray opening.

Sensor cable in its own channel, never bundled with mains or LED DC. Crossings at 90°.

\Alatarv aehamictru CANRCAFES AKA NAVAF AK tha ranLl nu and EC nrahac naad rat tina nalihratinn
```

## Page 10
```
~ WVQAULG! VIIVGIINO y VUIIVVII GIT LIVVUG! VII LUItG FauUnNs Vv! 1am uw Vv! VYNGEOHICTCU I PVYVULITIC LVANNIAGLVIT

and periodic replacement; putting them at the reservoir means one calibration for the whole

room instead of one per rack.

05 - VENTILATION

 

 

 

 

 

 

 

 

 

 

 

Air through the rack
AIRFLOW — RACK SECTION MULTI-RACK ROOM CIRCULATION
rear plenum - full-length perforated face - 178 m*/h per tier - @.2-0.5 m/s at canopy rears face rears - fronts face aisles - no supply-return short circuit
f <4 ROOM PLAN
| <q] ©
TIER 4 BED
| BA ee <q fa SUPPLY — cooled,
| it
a 5
| a
PT eee ee eee eee =
<4 eS
| <= ©) AISLE REAR-TO-REAR SERVICE CORRIDOR wi
| < 0 +
| TIER 3 BED 2 oO
| S
3 =
= ana Rane > RETURN — warm, md
S
| wo
ce
<
| <q
| =
TIER 2 BED >
4 Room HVAC owns temperature, humidity and C02.
a
I Rack fans own canopy air movement. Different jobs.
|
|
| <—+— ©
|
| TIER 1 BED
|
|

 

 

 

OPEN FRONT — air exits to the aisle and returns to room HVAC

RK-A-AQ1 + AIRFLOW SCHEMATIC - RACK SECTION AND ROOM CIRCULATION

~ >

Rack fans or room HVAC — the answer is both, doing different jobs

These are not alternatives. Room HVAC owns bulk temperature, humidity and CO>. Rack fans
own air movement at the canopy. Room HVAC physically cannot deliver 0.2-0.5 m/s to sixteen
canopies inside four enclosed tiers without ducting every tier of every rack and a fan far larger
than the sum of the small ones — moving air 6(0O mm across a canopy takes about 5 W,
whereas pushing conditioned room air through the same path takes an order of magnitude

more.

For the initial Ooty stage, rack-level EC fans are both the efficient and the feasible choice: 20
W per rack, a bolt-on part with no room interface, and it lets you commission the racks before

the room's HVAC is finalised. The room side then only has to hold air temperature and RH, which

nA nAKNAntinnal anlit unit Aliwa Anh imiAdnifons Anne
```

## Page 11
```
OC LVIIVOETILIVIIG! SMI ULL VIUS USHIUIINUNIOC! UUs.

PARAMETER

Fan type

Duty required

Free-air rating

to specify

Bearings

Ingress

Voltage / power

Mounting

Perforation

Direction

Control

SPECIFICATION

EC / DC axial, 120 « 38 mm

178 m?/h at = 19 Pa

> 254 m3/h

Ball, not sleeve

IP54 minimum

24 V DC, = 5 W each

Plenum end cap, right-hand end of
each tier, anti-vibration grommets

= 1590 x 04 mm at 16 mm pitch, 200

cm?

open
Rear plenum > forward across the
canopy > open front + aisle > room
return

PWM or 0-10 V from the rack
controller, with tacho feedback

BASIS

EC for speed control and low power; 38

mm depth for static pressure

0.35 m/s over the 0141 m? canopy
cross-section

A fan ona perforated plenum delivers
~10% of free air

Sleeve bearings fail early in high
humidity and cannot run horizontally
for long

Condensing environment

Same 24 V rail as solenoids and
sensors

Off the structure, so fan vibration does
not enter the frame

Face velocity 2.5 m/s, perforation loss 9
Pa

Horizontal, entering along the full 1176
mm

Tacho is how a failed fan raises an
alarm before the crop notices

SD >

Preventing stagnation and microclimates

¢ Distributed inlet, not a point source. This is the whole reason for the plenum: supplying

along the full shelf length rather than from one end takes the velocity coefficient of variation

from roughly 33% to about 10% in the published CFD work. A clip-on fan at one end of a shelf

is the arrangement that evidence specifically rejects.

e Fansrun continuously, including through the dark period. Stagnation at night is when

condensation forms on leaves and disease starts.

e Every tier has its own fan. Tier-to-tier microclimates are prevented by giving each tier its

moan Anns entlas thaw DAwniAn nN Ale GAARA itn ssn,
```

## Page 12
```
OWTI SUPPLY FAL! Uldl MOpPINIY dail HMIUS Ils Way.

¢ The alarm case matters more than the design case. A tier whose fan has failed looks
identical to a working one until the crop shows it. Tacho feedback and a controller alarm are
not optional extras here.

e With side panels deleted, verify uniformity by measurement at commissioning — see §OO.

Interaction with room HVAC and dehumidification. Every watt of LED energy that does not
leave as photons leaves as heat, and all of the irrigation water that does not leave in the crop
leaves as vapour. The rack fans do not remove either — they only move air across the canopy so
the leaf boundary layer stays thin. The room HVAC and dehumidifier carry the whole thermal
and moisture load. Sizing that is Phase 2 work; what the rack must do, and does, is present a

predictable return air condition to the room rather than a set of stagnant pockets.

 

06 - LED MOUNTING
A mount, not an integration

The objective is that a fixture is a consumable and the rack is not. Everything below
follows from that.

ELEMENT SPECIFICATION
Rail RK-A-301, aluminium 20 x 20 x 1.5, 1176 mm, two per tier at 30% and 70% of bed depth

Rail attachment Gl saddle bracket bolted to the 50 mm grid on the underside of the tier above, M8

S$S304, nylon isolating washer between aluminium and galvanised steel

Height Adjustable, continuously — M8 x 150 threaded-rod hangers through the rail with a nut

adjustment above and a wingnut below, giving O-100 mm of trim under a rail that sits on the discrete
grid

Why threaded Wire-rope hangers are faster but let fixtures swing, which makes PPFD uniformity

rod, not wire unrepeatable and lets a fixture drift after someone brushes it while harvesting

Fixture interface Two M8 clearance holes at 900 mm centres on the fixture top face. Product 1.2
conforms to this; any third-party bar that conforms will mount

Fixture envelope 1000-1200 mm long, < 80 mm wide, < 4kg

Electrical Fixture pigtail to an IP65 keyed connector at a fixed position per tier. A fixture change is
```

## Page 13
```
connection never a rewiring job

Driver location Remote, in the rack enclosure — not on the fixture. Keeps mass and heat out of the

canopy, lets one driver serve a tier, and makes a driver failure a two-minute swap

Supply to fixture Constant-voltage 48 V DC, or constant-current with output < 60 V. The tier stays

extra-low-voltage — no mains-voltage conductor enters the wet zone

Strain relief P-clips to the rail at 300 mm centres, plus a150 mm service loop at the fixture end soa

fixture can be lowered for inspection without unplugging

Removal Undo two wingnuts, unplug one connector, lift out. No tools beyond a spanner, no
dismantling of frame, deck or plumbing

Future upgrade The interface is three things: the rail, the 90O mm bolt centres, and the connector. Any
path future fixture meeting those bolts on — including a higher-output bar for leafy greens,
which the 80-200 mm clearance band already accommodates

—

The ELV decision is the one that matters for safety. Putting the driver in the rack enclosure
and running 48 V DC to the fixture means the only mains conductors on the rack are inside an
IP65 box mounted above canopy height. Everything at or below bed level — where the water is
— is 24V or 48 V. That, rather than any ingress rating, is what makes the electrical and water
systems genuinely independent.

07 - ELECTRICAL AND CONTROL

Three levels

>

 

POWER — radial tree

amber - MDB + group panel + rack

 

RACK + IP65 ENCLOSURE, TOP REAR, DRY ZONE

 

 

MAIN DB GROUP PANEL RCBO 16 A

SPD Type 1+2 Pcs vices - spp 2 tye a - 30 ma | | ISOLATOR 24 V PSU | | LED DRIVER

per 8 racks one per rack lockable, LOTO 48 V out

 

 

 

 

 

 

 

 

RACK CONTROLLER - ESP32 + 8 CH RELAY + RS485
fatve sequencing - fan PWM + tacho - leak interlock - local fail-s|

8 = VALVE
fill + 4 drail
4 SEGREGATED CHANNELS:

— 230 V mains — 48 V LED DC
— 0-10 V dim — RS485 + sensors

 

 

o

 

 

4 = FAN T/RH + LEAK

 

 

 

 

 

4 «x LED
48 V + 0-10 V

 

frame bonded 4 mm? + continuity < 0.1 0 - glands downward - dip Loops

 

CONTROL — daisy-chained bus
blue + RS485 multi-drop, never a star
MQTT over TLS

ROOM CONTROLLER WATER SKID .
local DB. Local UI RACK 02 RACK 02 RACK 03 RACK n<32 4 ow Ee we row | ROOM 1/0 L CLOUD, INDIA REGION

 

 

 
```

## Page 14
```
1200 120 0

vu
DASHBOARD / APP

cloud is optional — the room runs

RK-A-E01 - ELECTRICAL AND CONTROL ARCHITECTURE - POWER RADIAL, CONTROL MULTI-DROP

~ >

Your proposed topology, with one correction

You proposed Rack + Rack Controller + Group Control Panel + Main Distribution Board. Thatis

right for power, and | would keep it exactly as stated. It is wrong for control, and the distinction

is worth making because getting it backwards is acommon and expensive mistake.

Power is a radial tree Control is a multi-drop bus

MDB - group panel (one per eight racks, 63 A) RS485 is a bus. Wiring it as a star through a

— per-rack 16 A RCBO >= rack. Branches never group panel creates stub reflections and an
rejoin. This is your architecture and it is correct. unreliable link. The rack controllers daisy-chain

along one cable, terminated 120 Q at each
physical end, with the room controller at one
end.

So the group panel is a power object, not a control object. Control skips it entirely and runs as

one chain from the room controller through every rack.

Level 1 — the individual rack

ITEM

Power input

Enclosure

Local isolation

Protection

Distribution

LED

connections

Fan

SPECIFICATION
230 V single phase, 1.37 A, via gland into the rack enclosure. Total connected load 316 W

IP65 polycarbonate 300 x 200 x 150, mounted top-rear on the services rail — above

canopy, in the dry zone, glands facing downward

Lockable 16 A DP rotary isolator on the enclosure door, for lock-out/tag-out

16 A Type A RCBO, 30 mA, at the group panel — one per rack

230 V > LED driver (48 V out) and 24 V DC PSU. 24 V = fans, solenoids, controller, sensors

48 V DC + 0-10 V dimming pair per tier, IP65 keyed connector at the tier

24 VV + PWM + tacho, 3-core to each plenum end cap
```

## Page 15
```
connections

Valve
connections

Sensor

connections

Controller

Emergency
shut-off

24 V switched by relay, eight circuits. Valves are the only electrical items below bed level,

and they are 24 V

I?C to the T/RH node, dry contact from the leak sensor, in the dedicated channel

ESP32-class with 8-channel relay and RS485 transceiver. Runs the tier sequence locally

and holds last-known-good setpoints, so the rack keeps irrigating if the bus or room

controller drops

Room-level E-stop, not rack-level. Latching mushroom head at each door, dropping the

group contactor. A separate E-stop on each of sixteen racks is sixteen devices nobody can

reach in an emergency; the rack isolator is for maintenance, which is a different job

~ ee >

Level 3 — sensors and communication

 

QUESTION ANSWER WHY
Sensor to °C for digital T/RH (short No analogue runs longer than the rack
controller run, inside the rack); dry
contact for leak
Wired or Wired A room of galvanised racks is a poor 2.4 GHz environment,
wireless the cable route already exists for power, and Phase A calls for
local-first reliability. Wireless stays available as a retrofit for a
rack that cannot be cabled
Protocol Modbus RTU over RS485, Multi-drop to 32 nodes, 1200 m, immune to the electrical
shielded twisted pair, 120 Q noise of LED drivers, and every PLC and gateway on the
terminated both ends market speaks it
Controller Daisy chain, one address Adding a rack is a spur and an address, not a new panel
to central per rack
Data Room controller polls the Local-first per Phase A §2.5 — the room keeps running and
collection bus, writes to a local time- keeps logging with no internet
series database
Cloud and MOTT over TLS to an Optional layer. Nothing in the growing process depends on it
app India-region endpoint,

dashboard and alerts on
top

Se >
```

## Page 16
```
Level 4 — protection and safety

MEASURE

Per rack

Per group

Main

Earthing

Water ingress

Cable
management

Emergency

isolation

Interlocks

SPECIFICATION

16 A Type ARCBO, 30 mA. Type A rather than AC because LED drivers and PWM fans
produce pulsating DC residual currents that an AC-type device can miss

63 A MCCB, Type 2 SPD, one group per eight racks
Type 1+2 SPD at the MDB, main earth bar

TN-S per IS 3043. Every rack frame bonded with 4 mm* green/yellow to the group earth
bar. Continuity < 0.1 QO, tested and recorded per rack

IP65 at and below bed level, IP54 above canopy. All glands face downward. A drip loop
on every cable entering an enclosure

Four segregated channels. Minimum 50 mm between mains and data, crossings at 90°
Room E-stop at each door, latching, dropping the group contactor. Rack isolator for
LOTO

Leak sensor closes the rack's fill solenoids directly through the controller, and raises an

alarm. This runs locally and does not depend on the bus

(ee >

08

BILL OF MATERIALS

Costed, panels removed

SUBSYSTEM

Core rack

Flood trays x 4

Irrigation

Drainage

KEY CONTENT COST

Frame, decks, feet, anchors — unchanged from RK-A-MFG 312,391
HDPE 3 mm food grade, white 4,835
4 fill + 4 drain solenoids, 8 bulkheads, 4 standpipes, strainers, isolation %7,650

valve, PE pipework

DN40 header, clamp couplings, air-gap discharge %1, 300
```

## Page 17
```
 

Ventilation

Plenum face

OPTIONAL

LED mounting

Electrical & control

Rack sensors

Powder coat

Rev 2 changes

GROW — full
automation

GROW-S — de-
scoped

4 EC fans, 4 perforated plenums, grommets, brackets

Plain perforated white PP x 4 — enclosure and reflector only; the Gl

plenum already distributes the air

8 rails, 16 hangers, 16 saddles, 4 P65 connector pairs — fixtures not

included

IP65 enclosure, isolator, RCBO, 24 V PSU, rack controller, segregated
cabling, bonding

T/RH node in radiation shield, leak sensor

Optional on Grow

Header level switch, moulded overflow collar, grouped fill manifold

1 fill solenoid, single header drain valve, no rack controller, room T/RH

only, mill Gl

5,520

TO - +
=600

5, 460

9,580

1,650

%2,700

+3170

351,256

%43,286

Se >

BUILD COST

Grow- 343,286

Grow %51,256

@ 40% GM @ 45% GM USE

72,150 78,700 Build this at Ooty. The room controller drives two racks

directly — a per-rack controller earns its place at eight

racks, not two

%85, 400 93,280 = Commercial standard once rack count justifies distributed

control

CD >

Read this honestly. A fully automated four-tier flood-and-drain rack with per-tier lighting,

airflow and valving is a 43,000-51,000 object, and no amount of value engineering makes it a

=10,000 one. The wire shelf it gets compared to is a shelf. This is a growing system with sixteen

irrigation zones, four climate zones and a controller. The comparison that matters is against

importing an equivalent automated system, not against a shelf — and that is the argument

the product has to be sold on. If it must be sold against a shelf, the honest answer is to sell the

Core rack at 20,650 and let the customer add subsystems as they grow, which is exactly what

the modular architecture is for.
```

## Page 18
```
09 - ROUTING

Where everything runs

ZONE CONTENTS RULE

Rear service zone, left DN20 supply riser, DN16 Water only. Clipped to the grid at 400 mm

upright tier drops

Rear service zone, DN40 drain header, leak Water only, opposite side from supply soa

right upright sensor cable leak is unambiguous

Rear plenum cavity Fan cable, 24 V + PWM + Low voltage only, grommeted through the
tacho plenum wall

Top rear, above IP65 enclosure, mains, 48 V Dry zone. Nothing at mains potential exists

canopy DC, RS485 below this level

Front-right upright T/RH node, sensor cable Clear of the 1176 mm tray opening

face

Under each tier 48 V + 0-10 V to the LED In the LED rail P-clips, 150 mm service loop at
connector the fixture

Front opening Nothing The full 1176 x 343 opening stays clear for

tray handling

CO >

10 - VALIDATION
Failure modes and what they cost

Ranked by what actually goes wrong in wet, automated growing equipment rather than

by what is easy to analyse.

# FAILURE MODE EFFECT MITIGATION IN THE DESIGN STATUS

Fi Tray drain Continuous leak onto EPDM washer on the wet face, COVERED

bulkhead seal the tier below, then torque spec, 30-min flood test on
```

## Page 19
```
F2

F3

F4

F5

me)

F7

F8

F9

fails

Fill solenoid
sticks open

Drain boss
blocks with

media or roots

Drain header
blocks below
both bosses

Fan fails silently

Galvanic
corrosion, SS304
fastener in
galvanised steel

Galvanic
corrosion,
aluminium LED
rail against
galvanised
saddle

Standpipe not
refitted after

cleaning

Bus or room
controller lost

the floor. Crop loss on
two tiers

Would overflow the
tray rim

Tray cannot empty

Both drain and
overflow defeated.
Tray fills to rim then

spills

Stagnant tier,
condensation, tip-
burn and disease.
Looks identical to a
working tier until the

crop shows it

Zinc is anodic to
stainless and
sacrifices at the
washer face. White
rust, then red rust, at
every bolted joint in
the wet zone

Aluminium is anodic
to zinc-coated steel
in condensate. Pitting
at the bracket

Tray will not hold a
flood. Water runs
straight through and
the crop is not

irrigated, silently

Racks stop irrigating

100% of trays, leak sensor and
solenoid interlock

Overflow standpipe at 30 mm — COVERED
passive, no moving parts. Water goes

to drain, not to the floor

Removable mesh strainer, plus the COVERED
overflow on an independent
bulkhead so a blocked drain still has a

path

Rev 2: header high-level switch at COVERED
200 mm closes all fills before any tray
is affected. Rodding eye at the base.

See R1

Tacho feedback per fan, controller COVERED

alarm on loss of rotation

Nylon isolating washer under every COVERED
SS fastener at or below bed level.

Small stainless area against large zinc

area keeps the rate low, but the

washer face is where it shows

Nylon isolating washer at every rail- COVERED
to-saddle joint. This is a new finding
from this pass — it was notin the

earlier BOM and is now aline item

Rev 2: eliminated. The overflow is ELIMINATED
now a30 mmcollar moulded into the
tray floor. Nothing to remove, nothing

to lose. See R2

Rack controller holds last-known- COVERED

good schedule and runs locally. Loss
```

## Page 20
```
F198 Condensate None — it is water
drips from fixture going where water
into tray already goes

F11l ~—- Nutrient back- Contamination of the
siphonage into whole room's water
supply main

F12 ~~ Residual current An AC-type RCD may

from LED drivers not trip on pulsating

DC residual current

of busis an alarm, not a stoppage

No action needed. Noted so nobody ACCEPTED
designs a drip tray for it

Physical air gap at all 16 fill points. COVERED
Prevented by geometry, not by a

check valve

Type A RCBO specified, not Type AC COVERED

~

Open items carried forward

ITEM

Depth-plane bracing — still open from the structural pass

Canopy velocity uniformity without side panels

CLOSES BY

Prototype sway test P3

>

Measure at commissioning, 5 points per tier

~

Manufacturing and maintenance review

Manufacturing

Nothing added in this pass needs a process the
shop does not already have. The plenum isa
folded, perforated GI part — the same sheet
operation as the deck. The bulkheads, valves
and pipe are bought-out. The one new
tolerance that matters is the tray drain and
overflow boss positions, which must match the

header spacing across all four trays.

Water/electrical safety

The strongest feature is that only 24 V exists at
or below bed level. Mains is confined to one

IN@E hRAw ARAIAR RAWAL: Thaw

innleant waint ian

Maintenance

Trays lift out upward after releasing one clamp
coupling. The drain stack lifts out as one piece.
Fixtures come off with two wingnuts and a
connector. Fans are in the plenum end cap,
reachable from the aisle. The one poor-access
item is the fill solenoid at the top tier, at 1825
mm — reachable, but a two-hand job at head
height. Consider grouping all four fill solenoids
ona single serviceable manifold at 1200 mm

instead of one at each tier.

Areas needing redesign

Both closed in Rev 2. The fill solenoid position
is resolved by the grouped manifold (R3) and

than AviArflAias eAitaA Ais than WanAane lAwAl avuasitnlh

>
```

## Page 21
```
IT UOVU VUAK aVUVE UdIIUpPy. LIE WEdKESL VUITILIS UIle OVETMOW TOULS Vy LII€ HeAUC! ISVEl OWILUTI
not electrical at all — it is F4, a blocked header, plus the moulded collar (R1, R2). What remains
where the only defence is detection. are two tests, not two design questions.

11 - RESOLUTIONS
Closing the open items

Three of the four are now closed by design change rather than by procedure. The
fourth needs a test. One of these resolutions removes a failure mode entirely rather
than mitigating it, which is always the better outcome.

R1:F4 — blocked drain header

The insight that resolves this is where the water goes when the header blocks. Trays drain into
the header at their own tier heights, so a blockage below tier 1 causes the level to rise from the
base and re-enter tier 7's tray first, through its own drain line. That makes the header itself the
earliest and most certain place to detect the fault — well before any tray is at risk.

MEASURE SPECIFICATION

Header high-level Float or optical level switch in the DN40 header at 200 mm, just below tier 1's drain entry
switch

Interlock action Closes all four fill solenoids. Does not open the drains — draining into a blocked header

makes the situation worse, not better. Raises an alarm
Wiring Hard-wired to the rack controller as a local interlock, independent of the RS485 bus

Access Rodding eye at the header base and a removable top cap, so the stack can be cleared
without dismantling

Cost +%630

< Se >

F4 moves from restpuat to covered . [he detection is upstream of the consequence rather than
after it.

R2:F8 — standpipe left out after cleaning
```

## Page 22
```
Rev 1 mitigated this with a tether and a checklist, which | said at the time were weak controls.
They are. The better answer is to delete the removable part.

The overflow becomes a moulded collar, not a loose pipe. Thermoform a 30 mm raised collar
around the overflow hole in the tray floor. The collar isthe standpipe, formed into the tray,
permanently. Water above 30 mm spills over the crest and down the bulkhead. It costs nothing
in tooling — the form is already being made — and it cannot be lost, left out, or fitted upside

down. For the fabricated prototype trays, the equivalent is a welded-in fixed tube.

F8 is eliminated, not mitigated. Four loose standpipes come out of the BOM at -%760.

 

Collar bore stays DN32 minimum. At 8 mm over the crest, DN25 passes exactly the 7.2 L/min fill
rate with no margin; DN32 passes 11.9 L/min, a 1.65x margin.
```

