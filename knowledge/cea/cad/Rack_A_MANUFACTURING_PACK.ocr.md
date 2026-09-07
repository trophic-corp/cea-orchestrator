## Page 01
```
DOC RK-A-MFG - REV 2 - 05 SEP 2026 - SYSTEMS-INTEGRATED

RACK A
MANUFACTURING PACK

Modular CEA growing rack, 4-tier Grow configuration. Everything a fabricator, an
assembler and a QC inspector needs to build and accept one unit. Derived from
Fusion model cEA_RACK_INTEGRATED_v2 at Rev 5 — the systems-integrated rack,
interference-clean. Structural drawings are issued separately as RK-A-DWG Rev 2;
validation is recorded in RK-A-QC Rev 2.

INSTALLED TIERS MASS, COMPLETE PARTS IN MODEL FASTENERS

1456 = 690 x 4-16 trays 112.7 kg 45 unique 208 structural
1960

VALIDATION
26 checks - 35

closed

O01 - SPECIFICATION

FINAL RACK SPECIFICATION

PARAMETER VALUE BASIS
Installed 1456 W x 690 D x 1960 H mm Includes end-mounted electrical enclosure and rear
envelope plenum

Envelope, 1256 x 563 x 1960 mm Frame, decks and trays — the set drawn in RK-A-
structure only DWG Rev 2

Envelope, Phase 1 1456 x 648 x 1960 mm Plenum deferred; still set the room out at 690

build

Structural 1256 x 560 mm Upright outer faces

footprint

Tiers 4 (structure supports 3-5) Uprights drilled for 5
```

## Page 02
```
Bed heights

Shelf pitch

Bed size

Tray standard

Tray capacity

Rated shelf load

Bed flatness
under load

Irrigation

Supply / tray drain
/ header

Overflow

Wet-services
corridor

Valve fail-safe

Flood volume

Lighting

Airflow

Electrical

Floor clearance

Frame material

Accessory
interface

300 - 700 - 1100 - 1500 mm
400 mm
1176 x 560 mm

1020, band 530-545 x 275-
285 x 55-70

16 trays

80 kg UDL per tier

< 3 mm; 0.50 mm in service

Flood and drain, closed
loop

DN25 / DN40 / DN50

30 mm collar moulded into
the tray, DN32 line

Y 435-525 mm from the
front face

Fill normally CLOSED -
drain normally OPEN

14.5 L per tier

2 bars per tier, 144 mm to
canopy

178 m?/h per tier, 0.2-0.5
m/s canopy

230 V 1-ph, 316 W, 1.37 A,
30 mA Type A RCBO

150 mm under lowest member

Pre-galvanised steel, mill
finish

50 mm hole grid, 89, two
faces

Tray support surface

350 in the 5-tier build

4 x 1020 trays per tier

Design tray 536 x 279 x 64

4 per tier

Governing service case 38.0 kg

Flood-and-drain levelness

Flood 22 mm, dwell 10 min. Gravity supply from a

terrace reservoir; RK-A-WRS Rev 3

Drain always exceeds supply; header sized for
sequential tier draining

Sets flood depth mechanically. 11.9 L/min against
7.2 L/min fill

Forward of the rear bracing plane, clear of both LED
rows

A power cut drains every bed and fills none

Freeboard 16.5 L

PPFD band 150-210 pmol-m ?:s ?

Linear rear plenum

One feed per rack, local isolator. Type A, not AC —
LED drivers give pulsating DC residual current

Cleanability

Powder coat optional

36 positions per upright face
```

## Page 03
```
02

ENGINEERING VALIDATION

FIFTEEN CHECKS

Every line is acalculation, not an opinion. Two items are not PASS and are called out below

the table. No FEA has been run — these are closed-form calculations against the as-

modelled geometry, which is appropriate for a determinate bolted frame at this scale but is

stated plainly so nobody mistakes it for simulation.

Sd

10

11

CHECK

Static structural load — beam
bending at 80 kg UDL

Shelf deflection, service /
fault / design

Upright loading —
compression and buckling

Joint strength — 2 x M8 A2-
70 per bracket

Rack stability — seismic Zone
III

Tip-over — horizontal pull at
top bed

Water loading — blocked
drain

Equipment loading — LED rail
with 2 kg fixture

LED heat clearance

Fan vibration — beam mode
vs fan orders

Drainage — DN40 tray outlet

RESULT

o = 37.3 MPa, SF 5.6
0.50 / 0.85 / 1.79 mm
2.31 MPa (SF 91); Pcr

746 kN (SF 1313)

shear SF 157, bearing
SF 108

74 N-m vs 489 N-m

restoring

tips at 139 N empty,
307 N loaded

38.0 kg tier; 16.5 L
freeboard, 1.8 min

0.93 mm over 1176 mm
144 mm; air rise 1.0
K at 178 m?/h

13.3 Hz vs 33.3 / 233
Hz

30.7 L/min vs 7.2

of read aon

CRITERION

fy 210 MPa
(YST210)

< 3 mm flatness

fy, Euler at Le
400 mm

98 N per bolt

IS 1893, Ah 0.050

H:D 3.48:1

solenoid must

close < 60 s

visually flat

band 80-200 mm

no coincidence

drain > fill;

fe o4

VERDICT

PASS

PASS

PASS

PASS

PASS

ANCHOR

COMMISSION

PASS

PASS

PASS

PASS
```

## Page 04
```
 

under 22 mm head L/min Till, 4.3x sequential tiers

12 Operator access 343 mm opening; 560 NIOSH LI < 1.0 PASS
mm reach; LI 0.50-
0.72
13 Cleaning access — floor 150 mm (was 90, 150 mm target PASS
clearance corrected)
14 Electrical / water separation 236 mm vertical, no shared route PASS

IP65, 4 channels

15 Manufacturability — saw, punch, bolt; Coimbatore PASS
processes and yield 2.5-12% waste in 3s capability

Check 6 — tip-over. Fit the wall anchors. An empty rack tips under a 139 N horizontal pull at the
top bed — about 14 kg of force, which a person yanking a stuck tray will exceed without trying.
Loaded it takes 307 N. Height-to-depth is 3.48:1, below the 4:1 threshold at which anchoring is
normally mandated, so it is defensible as freestanding — but the empty case is genuinely marginal
and the fix is trivial. RK-A-106 wall anchors are included in the BOM and are to be fitted, not
optional. If a rack must stand free, the top tier is to be left unloaded and the rack labelled
accordingly.

Check 7 — the overflow interlock is a control specification, not a structural one. If the drain
blocks with the supply running, the bed reaches the tray rim in 1.8 minutes. The tray contains it —
no water reaches the floor — but only if the solenoid closes. The leak/level sensor and solenoid
must be proven to shut off within 60 seconds during commissioning, deliberately, with a blocked
drain. Until that test is signed off this is NEEDS VALIDATION .

Also still open from the design phase: the depth-plane bracing. Removing the per-tier cross
beams and relying on the decks as horizontal diaphragms plus the base frame is sound in principle
and the tip-over numbers above assume it holds. It has not been checked by frame analysis. If the
prototype shows racking under acceptance test T16, the fix is to restore the per-tier cross beams
at 562.

These fifteen checks are the structural set only. Systems integration added eleven more
validation areas and produced twenty-three Stage 1 findings, all closed — including four criticals
that no structural check could have found: a rear brace dimensioned 452 mm short, an overflow
line with a 90 mm break in it, four tiers whose simultaneous drain exceeded the stack, and valve
fail-safe polarity that had never been specified. The full register, with corrections and verification
method for each, is RK-A-QC Rev 2. Read it before quoting this pack.
```

## Page 05
```
03 - GENERAL ARRANGEMENT

GA DRAWING

 

FRONT ELEVATION
A-TIER GROW -

SCALE 1:1 IN MM

 

 

 

 

 

 

1950 RACK HEIGHT

 

 

 

40Q PITCH

 

 

 

 

 

 

300 .

15q

 

 

 

 

 

 

 

1256 OVERALL

 

1176 BED -

4 x 1020 TRAY

 

RK-A-000
RE-DRAWN

 

GENERAL ARRANGEMENT

ALL DIMENSIONS MM

DERIVED

SIDE ELEVATION

u

BED 4 - 1500
~~ BED 3 - 1100
~~ BED 2 - 700

BED 1 - 300

FET HAND -

AIRFLOW SHOWN —

 

 

 

 

 

 

144 LED+CANOP|

 

 

 

 

 

 

 

 

 

FROM THE FUSION MODEL,

560 BED DEPTH

570 OVERALL

NOT

 

 
```

## Page 06
```
04 - TIER SECTION

THE STACK, TIER BY TIER

This is the drawing that matters most on the shop floor, because every vertical dimension in

the rack is set by it. Datum is the bed — the tray support surface.

ep Ral So MegYE £3584%, +345

 

144)
345

 

 

 

LED FIXTURE + +283 to +323

CANOPY - +139

1020 TRAY 64 HIGH

MESH PANEL oafum ee FACE = DATL
DECK RAIL 25x25x1.5 - -—26.6 to -

BEAM 30x30x1.5 + -56.6 to —26.6

RK-A-001 - TIER SECTION - DIMENSIONS RELATIVE TO BED DATUM - 400 MM PITCH

The one rule a fabricator must not break: the mesh panel top face /s the bed datum. Everything

below it (deck rail, beam) hangs from that face, and everything above it (tray, canopy, LED) is

measured up from it. Build the deck to the datum, not to the beam.

05 - PART SCHEDULE

PART-BY-PART SPECIFICATION

PART NO. DESCRIPTION MATERIAL / SECTION CUT SIZE QTY
RK-A- Upright GI SHS 40x40x1.6 1950 4
101
RK-A- Beam, long GI SHS 30x30x1.5 1176 10
102

MASS PROCESS

3.65 Saw, punch 2
faces, deburr,
cap

1.58 Saw, punch ends,
deburr
```

## Page 07
```
RK-A-
103

RK-A-
104

RK-A-
105

RK-A-
106

RK-A-
107

RK-A-
201

RK-A-
202

RK-A-
203

RK-A-

301

RK-A-
401

RK-A-
402

RK-A-

403

RK-A-

404

RK-A-
501

Beam, short
(base)

Rear X-brace

pair

Levelling foot

Wall anchor
bracket

Beam-end L-
bracket

Deck rail, long

Deck rail, cross

Deck mesh

panel

LED mounting
rail
DRAWING HELD

Flood tray, wet
module

Tray drain
bulkhead

Tray strainer,
lift-out

Overflow
bulkhead

Plenum face /
reflector
PHASE 2

GI SHS 30x30x1.5

GI flat 25x3

M12 nylon-base adj.

foot

GI plate 4 mm

GI plate 3 mm

GI SHS 25x25x1.5

GI SHS 25x25x1.5

GI expanded mesh

1.6, 76% open

Al 6061 SHS

20x20x1.5

HDPE 3 mm, natural
white

PP tank connector,
EPDM wet-face
washer

SS316 mesh dome

PP tank connector,
EPDM wet-face
washer

White PP 2 mm

480

1797 x 2

corrected

G50 x 60

60 x 60

50 x 50 x

40

1176

510

1176 x

560

1176

1176 x
560 x 47

DN4O

G40 x 22

DN32

1040 x
170

12

16

.64

.12

.16

15

.07

. 30

.56

98

.35

31

.04

.10

.03

.34

Saw, punch ends,

deburr

Laser or shear,
punch, deburr

Bought out
Laser, bend, slot

Laser, bend,
punch

Saw, punch,
deburr

Saw, punch,
deburr

Shear, edge fold,
rivet

Saw, drill, deburr

Thermoform —
collar and both
bosses formed in
the tool

Bought out

Bought out or
press-formed

Bought out

Shear, punch,
perforate —
pattern set by
```

## Page 08
```
the
commissioning
velocity traverse

RK-A- Reflective panel, White PP 2 mm 465 x 225 8 0.20 Shear, punch
502 side OPTIONAL

UPRIGHT HOLE PATTERN — RK-A-101

This is the accessory interface and the single most important feature in the rack. Get it wrong
and nothing else fits.

FEATURE SPECIFICATION

Hole diameter 09.0 +0.2 / -0.0 (M8 clearance)

Pitch 50.0 + 0.2 mm cumulative over any 400 mm

First hole 150 mm from the foot end

Count per face 36 (150 to 1900)

Faces Two adjacent faces, holes on the section centreline

Through condition Through both walls, in line, single hit

Edge distance 15.5 mm to face edge (min 1.5 x d = 12)

Burr Both sides, < 0.1 mm — a burr inside the tube is a water trap

Cumulative pitch, not incremental. Measure every hole from the datum end, never hole-to-hole.
A 0.2 mm error repeated 36 times is 7 mm at the top, and the four uprights will no longer accept a
common beam.

06 - BOM AND MATERIALS

BILL OF MATERIALS

ITEM PART NO. QTY MATERIAL SPECIFICATION SOURCE

Striictiural 1A1-1A2 14 Pra-oalvanicad ctaal hallaw cactinn ta TS 4992 lacal ctackad
```

## Page 09
```
sections

Deck sections 201-202

Expanded 203
mesh

Flat bar 104
Plate parts 106-107
Aluminium rail 301
Flood tray 401
Reflective 501-502
panels

Levelling feet 105

Fasteners -

au Pe GME vewu nun SECU WY aw Trey

grade YST210 min, 2275 coating

24 Pre-galvanised SHS 25x25x1.5, IS 4923

4 GI expanded metal, 1.6 mm, > 70% open area

2 GI flat 25 x 3 mm

14 GI sheet 3 mm and 4mm

8 Al 6061-T6 SHS 20x20x1.5, mill or clear anodised

4 HDPE 3 mm, food-contact grade, natural / white,
resin traceable

12 White PP sheet 2 mm, matte, > 85% diffuse
reflectance

4 M12 x 60 adjustable, nylon base, +20 mm

208 See 807

Local, stocked

Local

Local

Local laser

Local extruder

Thermoformer

Local

Bought out

Bought out

Two material clauses that are not negotiable. The flood tray must be food-contact grade with

documented resin traceability — it is the only part in the rack the crop touches. And every cut end

of galvanised section exposes bare steel: ends are either capped (uprights, RK-A-101) or sealed

with zinc-rich touch-up (all others) before assembly. This is exactly what commercial greenhouse

benching does and it is what makes mill-finish GI acceptable in a wet room.

07 - FASTENERS

FASTENER SCHEDULE

APPLICATION

Beam to upright, 24 joints x 2

Deck long rail to beam, 16 ends

Deck cross to long rail, 32 ends

FASTENER QTY
M8 x 20 hex, SS304 A2-70 + flange 48
nut

M8 x 20 hex, SS304 + flange nut 16
M8 x 20 hex, SS304 + flange nut 32

TORQUE

18 N-m

18 N-m

18 N-m
```

## Page 10
```
Rear brace to upright, 4 ends x
2

LED rail to grid, 8 rails x 2

Wall anchor to upright

Wall anchor to wall

Levelling feet

Deck mesh to deck frame

Reflective panels to grid

M8 x 20 hex, SS304 + flange nut

M8 x 20 hex, SS304 + flange nut

M8 x 20 hex, SS304 + flange nut

M8 masonry anchor, min 60 mm embed

M12 x 60 adjustable foot

4.8 mm blind rivet, SS

M6 x 16 + penny washer, SS304

16

32

48

18 N-m

12 N-m

18 N-m

per anchor
maker

hand + locknut

208 fasteners is high, and it is the honest number. All-stainless was a deliberate call — a rusting

bolt in a food room is not worth the %330 saved by mixing grades. But the count is a real

assembly-time cost, and it is the strongest argument for the v2 tooling: a formed keyhole beam-

end removes 48 M8, and a tabbed deck cross rail removes a further 32. That is 38% of the

fastener count gone, along with most of the assembly labour.

08 - CUT LIST

NESTED CUT LIST

Order material in batches of three racks. On a single rack, offcut waste runs 22-35%;

nested across three it falls to 2.5-12%, and the uprights alone go from 35% waste to 2.5%.

STOCK CUTS REQUIRED, 1 RACK BARS, 1 RACK WASTE

GI SHS 40x40x1.6, 4 x 1950

6m

GI SHS 30x30x1.5, 10 x 1176

6m

GI SHS 25x25x1.5, 8 x 1176 -

6m

AL SHS 20x20x1.5, 8 x 1176

2 35.0%

2 x 480 3 29.3%
16 x 510 4 26.8%
2 21.6%

BARS,

3 RACKS WASTE

4 2.5%
7 9.1%
10 12.276

5 5.9%
```

## Page 11
```
6m
GI flat 25x3 2 x 1797

12 brackets - 2
anchors

GI sheet 3 /4 mm

mill tolerance is common to the set.

 

09 - SUBSYSTEMS

SUBSYSTEM LAYOUTS

1 AQ . 1% 2 10.2%

nest - nest -

The brace length changed in Rev 2. RK-A-104 was cut at 1345 mm in Rev 1 and is 1797 mm —
the diagonal of a 1195 x 1342 opening. A 1345 mm brace cannot be fitted at all. It also improves
the nest: three racks now waste 10.2% of the flat bar instead of 32.8%.

Saw allowance: add 3 mm kerf per cut when issuing to the shop. The nesting above already
assumes it is absorbed in the waste figure. Cut all four uprights from the same bar batch so any

Every subsystem below is modelled in CEA_RACK_INTEGRATED_v2 Rev 5 and is interference-clean

against the structure. Positions are given from the rack origin: X along the width from the

left end, Y from the front face, Z from the floor.

LED mounting

Two RK-A-301 rails per tier at Y 168 and 372,
bolted to the underside of the tier above — not
to their own supports, which is what caused the
original 4 mm clash. Only the top tier gets a
dedicated rail. Fixtures hang on M8 threaded-
rod hangers giving continuous trim; as built, 144
mm to canopy. Drivers remote in the end
enclosure, 48 V DC to the tier, IP65 keyed

connector.

Ventilation — Phase 2

Folded 0.8 mm GI plenum, 1040 x 120 x 170 at

V RAR-ARQE tha fan mavina intn ite and ran and

Ventilation — Phase 1

Standalone EC fans. One 120 x 120 x 38 fan
per tier on a grid-mounted bracket at the rear-
right of the bed, 178 m3/h. Expect a canopy
velocity CV near 33% from a point source.
Traverse five points per tier at commissioning
— that measurement is the design input for the

Phase 2 plenum, not just an acceptance check.

Irrigation

DN25 riser on the left, X 57-83, fed by gravity

fram tha tarracra racarvnir Fill manifald at 7
```

## Page 12
```
PYVY YU, Ie TAIT rey bho Vr Cp Us

the perforated face RK-A-501 at Y 563-565.
Same air, distributed along the full 1176 mm, CV
about 10%. Bolts to the same grid holes the
Phase 1 bracket uses. Adds 42 mm to installed
depth — which is why the room is set out at 690
from the start.

Drainage

All wet services in one corridor at Y 435-525,
forward of the rear bracing plane. Each tray
drains through a DN4O bulkhead with a lift-out
strainer, down to a drain solenoid, across to the
DN50 vertical header, and out over a 100 mm
air gap into the tundish at 170 mm. The
overflow runs in parallel from the moulded 30
mm collar through its own DN32 bulkhead —
two independent penetrations, so a blocked
drain still has a path.

10 - FABRICATION

FABRICATION INSTRUCTIONS

PPUEEE GIR CLUOPPUGY PEQULIVUTI. FL run ae

1000-1300 with all four solenoids grouped at
working height. DN20 drop per tier to a nozzle
discharging above the tray rim — a physical air
gap at all sixteen fill points, so back-siphonage
is prevented by geometry, not by a check valve.
14.5 L per tier, 10 minute dwell.

Sensors and electrical

T/RH node on the right end face at Z 1200; leak
puck in the base pan; header high-level switch
at 250 mm. Electrical is all at the left end: IP65
enclosure at Z 1700-1900 with the isolator on
its front face, cable channel down the left end,
per-tier trays. Nothing at mains potential exists
below canopy — the tier is 24 V and 48 V only.

There is no welding in this rack. Every structural joint is bolted. That is deliberate — it removes

the need for pickling and passivation, removes weld distortion from a frame that must hold 3 mm

flatness, and lets a general fabrication shop build it. The only welding anywhere is the flood tray,

and only if it is fabricated from sheet rather than thermoformed.

01 CUT

All sections to the §08 cut list, cold saw or bandsaw. Square cut, +1.0 mm on length, +0.5°

on squareness. Do not flame cut galvanised section.

02 PUNCH THE UPRIGHTS

To the RK-A-101 pattern. Cumulative dimensioning from the foot end. Punch or laser; if

drilling, use a jig — freehand marking will not hold 0.2 mm cumulative.
```

## Page 13
```
03

04

05

06

07

08

09

10

PUNCH BEAM AND RAIL ENDS

Two @9 holes per end on the centreline, 25 mm and 75 mm from the end face.

LASER THE PLATE PARTS
RK-A-106 anchors and RK-A-107 brackets from 3 and 4 mm GI sheet. Bend to 90° + 0.5°.

DEBURR EVERYTHING

Both faces of every hole, every cut end. A burr inside a tube holds water and starts corrosion
from the inside where nobody sees it.

SEAL THE CUT ENDS

Zinc-rich touch-up on every sawn end. Press plastic caps into both ends of all four uprights.
This is what makes mill-finish GI viable in a humid room.

OPTIONAL POWDER COAT

Grow and Pro tiers only. Polyester powder over the galvanising, RAL to brand. Mask the 50
mm grid holes or ream after coating — a coated hole will not take an M8 cleanly.

DECK ASSEMBLY

Rivet the expanded mesh into the deck frame, mesh top face flush with the frame top face.
This face is the bed datum.

FLOOD TRAY

Thermoform. Three features are formed in the tool, not drilled on site: the 30 mm overflow
collar, the DN4O drain boss at 60 mm from the right end and 480 mm from the front edge,
and the DN32 overflow boss 60 mm from it on the same line. Form a 2-3 mm fall across the
floor toward the drain boss. If fabricated from sheet for the prototype, weld corners inside
and out and radius them.

FLOOD-TEST EVERY TRAY

To the rim, 30 minutes, before it leaves the shop. A leak found in the workshop costs nothing;
the same leak on a loaded rack costs a crop.
```

## Page 14
```
11 =:

ASSEMBLY

ASSEMBLY MANUAL

Two people, roughly 90 minutes for a first build, 45 once familiar. Tools: 13 mm spanner

and socket, 10 mm spanner, torque wrench, rivet gun, spirit level, tape.

01

02

03

04

05

06

07

LAY OUT THE TWO SIDE FRAMES

On a flat floor, lay one front and one rear upright with the grid faces inward. Bolt the base
short beam RK-A-103 between them at 150 mm, using the L-brackets. Repeat for the second
side. Do not torque yet.

STAND AND CONNECT

Stand both side frames. Bolt the two base long beams RK-A-102 front and rear at 150 mm.
The rack is now self-standing but loose. Second person required.

FIT THE TIER BEAMS

Working bottom to top, bolt a pair of long beams at each tier — beam top face at bed - 26.6
mm, i.e. holes at 250, 650, 1050, 1450 from the floor. Count grid holes from the bottom, do
not measure each one.

FIT THE REAR X-BRACE

RK-A-104, 1797 mm between hole centres 1757 — check the length against the opening
before you cut a batch. Bolted to the rear face of all four upright ends.

SQUARE THE RACK BEFORE TOROQUING:

measure both rear diagonals and adjust until they are within 3 mm of each other. The brace
locks in whatever squareness you leave it with.

TORQUE THE FRAME

All M8 to 18 N-m, working from the base upward. Re-check the diagonals afterwards.

FIT THE FEET AND LEVEL
Thread in the four RK-A-105 feet. Level across the width and the depth at the lowest tier.
LEVELNESS IS A FUNCTIONAL REQUIREMENT, NOT A NICETY

— an out-of-level bed gives uneven flood depth across the four trays.

FIT THE WALL ANCHORS
RK-A-106 to the rear uprights at 1870 mm. then to the wall. This is not optional — see
```

## Page 15
```
08

09

10

11

12

13

14

15

validation check 6.

DROP IN THE DECKS

One deck assembly per tier, resting on the tier beams, bolted at four corners. Check the deck
top face is level and sits at the nominal bed height.

FIT THE LED RAILS

Two per tier on the grid at bed + 325 mm. Hang fixtures and set height to 144 mm above
expected canopy, or to suit the crop.

FIT THE TRAY BULKHEADS
DN4O drain and DN32 overflow tank connectors,
EPDM WASHER ON THE WET FACE

— water pressure then seats the seal instead of lifting it. Drop the lift-out strainer into the
drain boss. Do not overtighten; HDPE creeps.

PLUMB

Riser and fill manifold on the left, all four fill solenoids grouped at 1100 mm. Drain header on
the right in the wet corridor at Y 435-525. Terminate over the tundish with the

100 MM AIR GAP

— never a direct connection.

WIRE

Enclosure on the left end at 1700 mm, cable channel down the left end, per-tier trays. Four
segregated channels: mains, 48 V LED, 0-10 V dim, RS485 and sensors. Keep every water

route below every electrical route.

FIT THE FANS

One per tier on the grid bracket at the rear-right of each bed. Phase 2 replaces the bracket
with the plenum end cap; the fan and its cable move across unaltered.

OPTIONAL PANELS

Side reflectors inboard of the uprights, M6 with penny washers, 6 N-m. The plenum face is a
Phase 2 part.

FLOOD TEST BEFORE PLANTING

Poll an ARK tink RAIA FAR Baten tann Aen RnR CRA AL FRE IAAL AR ARAL LIA RA AAR R44 IR RAR AR AeA RR AI!
```

## Page 16
```
Fill CACTI LICT, NOLU LET IMIMIIULES, UlAalll. UIIECK IO LEdaKs, CIIECK 1LOUU UCPLIT IS CVEIIACTUSS all

four tray positions, and confirm the drain empties in under a minute.

12 - QUALITY CONTROL

QC CHECKLIST

INCOMING AND IN-PROCESS

Section grade — mill certificate for IS 4923 YST210, Z275 coating, retained per batch
Cut length — every part measured, +1.0 mm; sample 100% on the first batch

Upright hole pattern — cumulative position from datum end, +0.2 mm over any 400 mm; check all 4

uprights of every rack

Hole diameter — 29.0 go / 9.3 no-go gauge

Deburr — visual and finger check both faces of every hole and every cut end

End sealing — zinc-rich touch-up present on every sawn end; caps fitted to all upright ends
Deck flatness — mesh top face flush with frame, < 1mm step

Flood tray — 30 minute flood test to rim, zero leakage, before despatch

Tray collar crest — 30 +0/-1 mm above the tray floor, measured at four points around the crest

Tray boss positions — centres 60 +1.0 mm apart and 480 +1.0 mm from the front edge, on all four
trays of a rack

Brace length — RK-A-104 at 1797 +1.5 mm, hole centres 1757 +1.0 mm REV 2 CHANGE

ASSEMBLED RACK — ACCEPTANCE

Squareness — rear diagonals within 3 mm of each other

Bed height — each bed within +3 mm of nominal 300 / 700 / 1100 / 1500
```

## Page 17
```
13

Bed level — < 1 mm/m in both axes, every tier, measured loaded if possible

Torque — sample 20% of M8 joints at 18 N-m, mark each checked bolt

Flood depth uniformity — flood one tier, measure depth at all four tray positions, spread < 3 mm

Drain time — full tier empties in under 90 seconds

Drain continuity — trace tray > strainer > bulkhead — drop - valve - lateral > header, joint by joint;
confirm the tundish air gap is 100 mm and is not plumbed closed

Fail-safe — cut power mid-flood: fill solenoids close, drain solenoids open, every bed empties
HOLD POINT

Mass — assembled dry rack 112.7 + 7 kg; outside that band the build differs from the model

Leak interlock — block the drain, confirm solenoid closes within 60 seconds HoLpD POINT

Earth continuity — < 0.1 9 from frame to earth terminal

RCD trip — 30 mA device trips within rated time

Wall anchor — fitted, both brackets, into sound substrate

Edges — no sharp edge or burr reachable by hand anywhere on the rack

Labels — part/serial label and rated load label fitted

PROTOTYPE TESTING

PROTOTYPE TEST PLAN

STAGE OBJECTIVE TEST PASS CRITERION

P41 Single deck Load one deck to 80 kg UDL for 24 Deflection < 3 mm, no permanent set after
h unload

P41 Flood levelness Flood one bed, measure depth at Spread < 3mm

4 tray positions
```

## Page 18
```
P2

P2

P3

P3

P4

P4

P4

P5

 

14

Single tier live

LED thermal

Two-tier frame

Tip-over

Full rack,

loaded

Fault case

Cleaning cycle

Production
prototype

COST AND PRICE

COSTED BUILD

ELEMENT

GI sections

Punching

LED + fan + irrigation for one full

crop cycle

Run 24 h continuous, log fixture
and air temperature

Sway test — 300 N horizontal at
top bed, front-back and side

Anchored and unanchored, 300 N
pull at top bed

All tiers at service load, 30 days

Block a drain with supply running

Full strip, wash, reassemble,
timed

Full crop trial, both Ooty crops

sign against are the T-numbers.

DETAIL

34.66 kg @ %100/kg

50 mm grid, end prep

Canopy 0.2-0.5 m/s; PPFD 150-210, CV <
15%; no leak

Air rise < 3 K; no LED derating

Residual deflection < 5 mm; this closes the

open depth-plane bracing item

Anchored: no lift. Unanchored: confirm the
139 N/307N calculation

No loosening; re-torque check within 10%

Solenoid closes < 60 s, zero water on floor

< 45 min for one person; no tool needed
for deck or tray removal

Yield and quality against the Phase A
targets; operator sign-off on handling

This plan is superseded by the eighteen acceptance tests in RK-A-QC Rev 2. Those are
numbered T1-1T18, carry numeric acceptance criteria and a stated frequency, and two of them
exist specifically to close Stage 1 open items: T16, the depth-plane sway test, replaces P3 and
closes the bracing question; T10, the canopy velocity traverse, closes the airflow uniformity item
and doubles as the design input for the Phase 2 plenum. The staged P1—P5 sequence above is
retained because it is a sensible order in which to build up to a full rack; the acceptance criteria to

COST

%3 ,467

%1,800
```

## Page 19
```
L-brackets

Fasteners

Feet and caps

Assembly labour

Frame subtotal

Decks x 4

CORE

Flood trays x 4

Tray fittings

LED mounting

Irrigation

Drainage

Ventilation
PHASE 1

Electrical and

control

Rack sensors

Powder coat

Rev 2 detail
changes

GROW — Phase 1
build

Plenum retrofit

GROW — complete

PRO

TIER

12 pcs laser

208 pcs, SS304

4 + end caps

bolted, no welding

rails 19.44 kg + mesh 7.93 kg + fab

frame + decks

HDPE, collar and bosses formed in the tool

4 x DN40 bulkhead, 4 x strainer, 4 x DN32 overflow connector

8 rails, 16 hangers, 16 saddles, 4 IP65 connector pairs

riser, manifold, 4 fill solenoids, 4 drain solenoids,
nozzles, isolation

DN50 header, laterals, tundish, air gap, level switch

4 EC fans + grid brackets — plenum deferred

IP65 enclosure, isolator, Type A RCBO, 24 V PSU, rack

controller, segregated cabling

T/RH node in radiation shield, leak puck, header level switch

optional, over GI

moulded collar, grouped fill manifold, level switch

as built at Ooty, plenum deferred

Phase 2, month 2-3 — 4 folded GI plenums + perforated faces

with ducted plenum

+ SS304 wet zone, + per-tier sensing

PRICE @ 40% GM PRICE @ 45% GM POSITION

840

71,444

%520

%1,000

9,071

%3,320

712,391

24,835

2680

35,460

%7,650

21,300

X2,880

%9 ,580

%1,650

%2,700

+2170

X49 ,296

+%2,640

51,936

756,996
```

## Page 20
```
Core 712,391 720 ,650 X22 ,500 Entry platform, germination rack

Grow, Phase R49 ,296 %82 ,160 %89 , 630 Flagship as built at Ooty, excludes
1 LED fixtures
```

