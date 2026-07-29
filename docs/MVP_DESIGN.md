# VillageSlop MVP Design

Status: product and technical scope baseline

Working title: VillageSlop

Target: mouse-and-keyboard desktop, online 1v1

Target match length: 12–20 minutes

## 1. Product definition

### One-sentence pitch

Two rival gods grow autonomous villages toward a contested World Heart, using
belief, direct divine intervention, and a shared wildfire-and-rain ecosystem to
become the first god to ascend.

### MVP promise

The MVP must prove one thing:

> Managing a village while physically intervening as a god creates an
> understandable and competitive 1v1 match.

This is a complete playable match, not a general-purpose engine demo. Two
players must be able to connect to a referee, grow villages, clash, use and
counter an environmental disaster, reach a result, and replay it.

### Product assumptions

- The camera presents a stylized 3D world from an elevated, freely movable
  perspective.
- Players see the whole map. There is no hidden simulation or fog of war in the
  MVP; gods are omniscient.
- The first client build runs on the development platform. The MVP is not
  complete until a match remains deterministic between macOS arm64 and Windows
  x64.
- One fixed, mirrored map is preferable to several unbalanced or procedural
  maps.
- Low-level libraries are compatible with a custom-engine goal. An existing
  game engine, gameplay framework, renderer, physics engine, ECS, navigation
  system, or shipped UI framework is not.

## 2. Design pillars

### 2.1 Feel like a god

The player acts through a visible divine hand, places structures, moves one
friendly subject at a time, casts miracles, and changes the environment. Menu
decisions must have an immediate expression in the world.

### 2.2 A village with agency

Villagers are subjects, not a collection of individually micro-managed RTS
units. They eat, select jobs, carry resources, construct, worship, flee danger,
and repair damage according to player-set priorities.

### 2.3 A weaponized ecosystem

Moisture, vegetation, wind, rain, and fire form one small but interacting
system. The same rain that protects a village also improves food regrowth. The
same fire that denies contested ground can escape into friendly territory.

### 2.4 Competitive clarity

Every loss must have a visible cause and a possible response. Miracles are
telegraphed, the central objective is always visible, influence has clear
borders, and deterministic weather is forecast rather than surprising players
with arbitrary outcomes.

## 3. The MVP match

The working rules and numbers below are data-driven initial tuning targets.
They are specific enough to implement and are expected to change through
playtesting.

### 3.1 Match setup

- Two players, each with one color and one village.
- One fixed, mirrored map approximately 192 by 128 simulation meters.
- One Temple and 12 villagers per player.
- Starting resources: 250 Food, 300 Wood, and 50 Belief.
- One World Heart at the center.
- Mirrored forests, food meadows, and defensible terrain around each starting
  area.
- Population cap: 40 total subjects per player, of which at most 8 may be
  Guards.
- Simulation tick: 20 Hz.
- Command lead: four ticks for both players, fixed for the match.

### 3.2 Match phases

1. **Establish, minutes 0–4**
   - Secure Food and Wood.
   - Add housing and a nearer Storehouse.
   - Decide whether villagers prioritize growth, construction, or worship.
2. **Reach, minutes 4–9**
   - Build a chain of Shrines toward the World Heart.
   - Train a small number of Guards.
   - Respond to the forecast dry season.
3. **Contest, minutes 9–16**
   - Fight for the World Heart.
   - Break or defend influence chains.
   - Use Rain and Ember to shape the battlefield.
4. **Resolve, by minute 20 unless tied**
   - A player reaches the Ascendancy target, or the hard time limit selects the
     higher score.
   - A tie enters a bounded one-minute presence-based sudden death.

These are desired rhythms, not scripted phase transitions. Players may rush,
defend, or invest in a larger economy.

### 3.3 Victory

The World Heart awakens at 8:00. It can be occupied earlier, but cannot be
controlled or produce Ascendancy before then.

The World Heart uses a ten-meter-radius ring. Eligible presence is one point
for a Villager and two for a Guard whose center lies inside or on the ring.
Dead, held, fleeing, or still-training subjects contribute nothing.

Control is recalculated at the end of every simulation tick. A player controls
the Heart when:

- the Heart's center is covered by that player's active influence;
- that player has at least one point of eligible presence; and
- that player's presence is strictly greater than the opponent's.

Otherwise it is neutral for that tick; previous ownership grants no hysteresis.
Ascendancy is stored in twentieths of a displayed point, so every controlled
tick adds one subpoint without fractional arithmetic. At 16:00 the announced
World Heart Surge doubles new scoring to two subpoints per tick, giving a
trailing player a visible late comeback route without changing earlier points.

An uncontested controller earns one Ascendancy point per second before the
Surge and two afterward. The first player to 240 points wins. At 20:00, after
that tick's ordinary scoring, the player with more points wins.

If the scores are tied, a 60-second sudden-death period begins. Influence is no
longer required during sudden death; the first player to maintain strictly
greater eligible presence in the ring for five continuous seconds wins. If
neither player does so by 21:00, the match is a draw. This guarantees
termination even if all remaining Wood or Shrines have been lost.

The Temple is not destructible in the MVP. Houses, Storehouses, Shrines, and
Barracks can be damaged and repaired. Having one primary victory condition
keeps the first balance problem tractable and prevents base-race victories from
bypassing the god-game systems.

### 3.4 Other match endings

- Surrender awards the match to the opponent immediately.
- When the server detects an interrupted connection, it pauses at the next
  confirmed tick and begins a 30-second grace period. If the existing
  authenticated session resumes, play continues; otherwise that player
  forfeits when the period expires. Rejoining from a new client process is not
  an MVP feature.
- A server failure, incompatible state, or second state mismatch after
  checkpoint recovery ends the match as a technical no-contest. It must save
  the command log, hashes, and latest valid checkpoint rather than inventing a
  competitive winner.

Network pauses stop simulation ticks, so the 20:00 and 21:00 limits measure
simulated match time rather than wall-clock time.

## 4. Player loop and control model

### 4.1 Repeating loop

1. Set village labor priorities.
2. Place buildings inside connected influence.
3. Gather Food and Wood while keeping subjects fed and housed.
4. Generate Belief through a healthy population or dedicated worship.
5. Extend influence with Shrines.
6. Choose between more economy, Guards, or miracles.
7. Contest the World Heart and disrupt the opponent's influence chain.
8. Read the environment, start or contain fire, and convert control into
   Ascendancy.

### 4.2 What the player controls directly

- Camera and divine hand
- Building placement and cancellation
- Village-wide labor priorities
- Villager production toggle
- Guard training
- Guard group move, attack-move, and defend orders
- Rain and Ember targeting
- Picking up and dropping one friendly subject

### 4.3 What villagers decide

- Which available job satisfies the current labor priorities
- Which eligible source or work site to use
- The deterministic path to that target
- When to eat
- Where to deliver carried resources
- When to flee fire or hostile Guards
- Which damaged friendly structure to repair
- Which nearby construction project to advance

The player may inspect an individual villager and see its need, job, target,
and current reason. The player cannot issue ordinary move or harvest commands
to civilians.

### 4.4 Divine hand

The hand is the signature physical interaction:

- It can lift one friendly villager or Guard.
- A held subject is removed from ordinary simulation interactions.
- Authoritative hand state contains the held entity, pickup origin, pickup tick,
  and cooldown end tick. Continuous cursor motion is cosmetic.
- It can be dropped only into connected, uncontested friendly influence.
- A successful drop intentionally teleports the subject to the nearest valid
  navigation cell at the quantized target; there is no simulated throw path.
- It has an eight-second cooldown after a successful drop.
- A subject may be held for at most ten seconds. An invalid drop leaves it held;
  timeout, surrender, or disconnect returns it to the nearest valid cell around
  its recorded pickup origin and starts the cooldown.
- It cannot throw objects, damage units through physics, or deform terrain in
  the MVP.
- Losing influence while holding does not destroy or strand the subject. A
  valid friendly drop or the guaranteed origin return remains available.

The motion is rendered continuously, but pickup and drop are validated,
tick-stamped simulation commands. A rate-limited, unreliable hand-pose message
lets the opponent render cosmetic motion; it never affects hashes or replay.

## 5. Economy and village simulation

### 5.1 Resources

| Resource | Produced by | Spent on | Strategic tension |
| --- | --- | --- | --- |
| Food | Gathering renewable food meadows | New villagers and Guards | Civilian growth versus military recruitment |
| Wood | Chopping finite tree clusters | All ordinary structures | Expansion versus resilience |
| Belief | Fed, housed, safe villagers; faster while worshipping | Shrines and miracles | Map control versus immediate intervention |
| Influence | Connected Temple and Shrine coverage | Spatial permission, not a stockpile | Safe growth versus a vulnerable chain |
| Ascendancy | Holding the World Heart | Victory score only | Forces interaction and ends stalemates |

Food, Wood, and Belief are player-global stockpiles. Starting resources are
already in those stockpiles. Gathered Food and Wood become spendable only when
a villager deposits its carried load at an active Temple or Storehouse.
Villagers periodically deduct Food from the global stockpile. Missing a meal
reduces work speed; prolonged hunger causes health loss. There is no separate
happiness, disease, age, family, tax, technology, or trade model in the MVP.

Belief is capped at 200. A fed, housed villager that is not fleeing produces a
small base amount; a Worshipper produces three times that amount while doing no
economic work. The Temple automatically queues one new villager for 50 Food
when production is enabled, housing is available, and its 20-second production
slot is idle. Food is deducted when production starts. The Temple has no
backlog beyond its active slot.

### 5.2 Labor priorities

The village panel exposes four weighted priorities:

- Gather Food
- Gather Wood
- Build and repair
- Worship

Idle villagers deterministically select from eligible work according to these
weights, distance, urgency, and stable entity ID tie-breaking. Players may lock
a maximum number of workers to a category, but not assign named civilians one
by one.

Priorities are integers from 0 to 100. Every ten ticks, the village scheduler
uses largest-remainder allocation to turn nonzero weights into desired worker
counts after honoring category maximums. Hunger, fire escape, and an existing
valid reservation override reassignment. Remaining work is ranked by the total
tuple `(urgency descending, category priority descending, path cost, target
ID)`, and villagers are considered in ascending ID order. If every priority is
zero, safe villagers remain idle and explain that state in the UI.

### 5.3 Buildings

| Building | Initial cost | Function |
| --- | ---: | --- |
| Temple | Starting structure | Village core, initial influence, initial storage/drop-off, 16 subject capacity, villager production |
| House | 100 Wood | Adds 8 subject population capacity, up to the cap |
| Storehouse | 150 Wood | Resource drop-off and local Food access |
| Shrine | 120 Wood + 50 Belief | Extends connected influence |
| Barracks | 200 Wood | Trains Guards |

Placement rules:

- The Temple projects a 30-meter radius. A completed Shrine projects a
  20-meter radius only when its center is covered by the Temple or another
  already-active Shrine connected to the Temple.
- Active Shrines are recomputed as a stable-ID breadth-first traversal whenever
  the graph changes. Enemy overlap does not deactivate coverage; a location
  covered by both players is contested.
- A new building's center must be inside active friendly influence and outside
  contested influence.
- Buildings cannot overlap blocked terrain, resources, other buildings, or the
  World Heart ring.
- Shrines become inactive when disconnected from the Temple through the
  influence graph.
- Each player may have at most six active or under-construction Shrines, and
  Shrine centers must be at least 12 meters apart.
- A destroyed or disconnected Shrine retracts its coverage immediately.
- Completed ordinary buildings keep functioning after influence retracts.
  Unfinished construction pauses until its site is valid again.
- Losing housing never kills subjects. It blocks new production until current
  population is again below capacity.
- A valid placement deducts and escrows its full listed cost atomically.
  Builders then advance the site without hauling construction materials.
- Cancelling an unfinished site refunds 75% of its escrow, rounded down.
  Destruction refunds nothing. An invalid placement spends nothing.

### 5.4 Units

#### Villager

- Gathers, carries, builds, repairs, worships, eats, and flees.
- Has one visible job and one visible need state.
- Avoids combat unless trapped.

#### Guard

- Trained at a Barracks for 75 Food and one free population slot over 20
  seconds.
- Each Barracks has one active training slot and no backlog. A valid
  `TrainGuard` deducts Food and reserves population immediately; destroying the
  Barracks cancels training, releases the slot, and refunds half the Food.
- Does not gather or worship.
- Belongs to one of at most two groups and receives only group-level orders.
- Automatically attacks the closest valid hostile in its stance and range.
- Can damage Guards and non-Temple structures. Civilians cannot be directly
  targeted and do not fight; they flee nearby hostile Guards.

There are no heroes, formations, projectiles, siege units, creatures, livestock,
or faction-specific units in the MVP.

## 6. Divine powers and disaster system

### 6.1 Rain

- Target: circular area whose center is covered by active friendly influence;
  contested overlap remains a valid target.
- Cost: 35 Belief.
- Independent cooldown: 20 seconds.
- Radius: 12 meters.
- Duration: 8 seconds.
- Adds ground moisture, extinguishes fire, slows ignition, and accelerates Food
  regrowth for a limited time.
- The target area is shown before confirmation.

### 6.2 Ember

- Target: one vegetated cell covered by active friendly influence; contested
  overlap remains a valid target.
- Cost: 60 Belief.
- Independent cooldown: 30 seconds.
- Telegraph: visible 2.5-second ignition marker.
- Starts a fire if local fuel and moisture allow it.
- Can be answered by Rain, evacuation toward an authored firebreak, or accepting
  local damage to spend Belief elsewhere.
- A semantically invalid cast spends no Belief and starts no cooldown.

If Rain and an Ember ignition resolve on the same tick, Rain changes moisture
and extinguishes existing fire first; Ember then evaluates whether the target
can ignite.

### 6.3 Environment

The authoritative environment is a 96-by-64 grid of two-meter cells with:

- fuel;
- moisture;
- fire intensity;
- a fixed north/south wind direction and strength, aligned with the mirror axis
  so it does not favor either starting side; and
- regrowth progress for Food cells.

Fire updates at 5 Hz inside the 20 Hz simulation. Every update reads only the
previous grid and writes a separate next grid before swapping them. Spread uses
a deterministic hash of match seed, cell ID, neighbor direction, and tick
rather than an iteration-dependent random stream. Burned vegetation changes
fuel and appearance but not navigation walkability in the MVP. Fire damages
subjects and structures, consumes fuel, and eventually burns out.

A dry season begins at 8:00, is forecast one minute in advance, and lasts three
minutes. It increases moisture loss, reduces Food regrowth, and increases
ignition likelihood equally for both players. At 9:00, a five-second warning
precedes paired lightning strikes on one authored mirrored pair of neutral
vegetation cells selected from the match seed. Those strikes supply a natural,
symmetrical ignition even if neither player casts Ember. This dry-season
wildfire is the MVP's sole natural-disaster event. Additional storms, floods,
earthquakes, tornadoes, erosion, and terrain deformation are later work.

## 7. Map and information

### 7.1 Fixed mirrored map

The MVP ships with one authored heightfield:

- west/east starting plateaus mirrored across the map's north/south centerline;
- three broad approaches to the central World Heart;
- mirrored resource quantities and walking distances;
- shallow elevation changes that improve readability without requiring cliffs,
  jumping, or multi-level navigation;
- firebreaks and open ground around each Temple; and
- enough neutral vegetation near the center for the environment system to
  matter.

The simulation uses a one-meter navigation grid. The rendered heightfield may
be smoother and more detailed, but it cannot alter authoritative walkability.

### 7.2 Information model

The complete map and all units are visible to both gods. This is intentional:

- it fits the omniscient god fantasy;
- it avoids hidden-state replication inside deterministic lockstep;
- it makes disasters and counterplay readable; and
- it removes fog-of-war and scouting from the first balance problem.

The user interface must always expose current influence, the opponent's
Ascendancy progress, the weather forecast, and telegraphed divine powers.

## 8. User experience

### 8.1 Required screens

- Title screen
- Server-address connect and ready lobby
- Match loading and synchronization screen
- In-match HUD and local settings overlay; it never pauses online simulation
- Victory/defeat summary

Accounts, public matchmaking, parties, rankings, spectators, and progression
are outside the MVP.

### 8.2 In-match HUD

- Top bar: Food, Wood, Belief, population, ping, and connection state
- Center objective: both Ascendancy scores and contest state
- Village panel: four labor priorities and current allocation
- Build palette: four constructed buildings
- Power palette: Hand state, Rain, and Ember with costs/cooldowns
- Context panel: selected subject, structure, resource, or environment cell
- World overlays: influence, valid placement, paths in debug mode, power
  telegraphs, fire risk, and weather forecast

There is no minimap in the MVP. Maximum zoom-out must show the fixed map,
objective, and active fire clearly; failure replaces decorative world detail
with stronger overlays rather than silently adding another UI surface.

### 8.3 Controls

- WASD or edge pan: move camera
- Mouse wheel: zoom
- Middle drag or Q/E: rotate
- Left press/release: context action or hand pickup/drop
- Right click: cancel or Guard group order
- Number keys: building and miracle shortcuts
- Space: focus the World Heart

Gesture recognition and physics-driven throwing are not part of the MVP.

### 8.4 Visual and audio direction

- Chunky low-poly subjects and structures
- Painted terrain colors with high-contrast influence borders
- Oversized hand cursor with clear contextual poses
- Warm, slightly exaggerated early-3D presentation
- Parchment-and-stone interface motifs without sacrificing readability
- Short villager barks for hunger, danger, construction, and worship
- Distinct audio signatures for Rain, Ember telegraph, fire spread, objective
  contest, and Ascendancy gain

Finished character animation, cinematic effects, voiced narration, and a
large asset pipeline are not MVP requirements. Primitive or low-detail meshes
are acceptable if state remains easy to read.

## 9. Technical architecture

### 9.1 Technology boundary

Recommended foundation:

- C++23
- CMake and CTest
- SDL3 for platform, window, input, audio, and GPU access
- SDL3 GPU API for a project-owned renderer over Metal, Vulkan, and D3D12
- GameNetworkingSockets for encrypted reliable/unreliable datagram transport
- xxHash for cross-platform, non-security state-digest primitives

SDL3 and GameNetworkingSockets are low-level boundary libraries, not game
engines. The project owns:

- simulation and entity storage;
- renderer and shaders;
- scene extraction;
- navigation and steering;
- villager and combat AI;
- gameplay networking protocol;
- snapshots, hashing, and replay;
- collision queries used by gameplay;
- shipped user interface;
- asset formats and tools; and
- all game rules and content data.

No general rigid-body physics library is required. MVP interactions use grid
queries, circles, heightfield sampling, and deterministic kinematic movement.

### 9.2 Process topology

The build produces:

- `villageslop_client`
- `villageslop_server`
- `villageslop_tests`

One player may launch a separate local server process for direct-connect
testing. The server has no renderer and runs the same simulation library as
clients.

The supported online envelope is deliberately narrow: an operator starts one
server for one match on a publicly reachable UDP address and port, and both
players enter that address. LAN and manual port-forwarded hosting also work.
The MVP does not promise NAT traversal, signaling, relay, room codes, automatic
server allocation, or public matchmaking. Its online acceptance test uses the
public server from two separate consumer networks.

```text
Client A commands ─┐
                   ├──> referee server ──> ordered tick bundle
Client B commands ─┘          │                    │
                              │                    ├──> Client A simulation
                              │                    └──> Client B simulation
                              └──> server simulation and validation
```

The referee server:

- binds one player slot to each accepted connection for the lifetime of the
  match;
- validates message shape, connection identity, sequence, and command rate on
  receipt;
- establishes the match seed and tick-zero state;
- orders commands for each tick;
- performs resource, ownership, target, cooldown, and other semantic validation
  when the target tick executes;
- runs the canonical simulation;
- publishes periodic hashes;
- retains recent checkpoints for resynchronization;
- appends tick bundles and scheduled digests to the replay as they are produced,
  flushing at least once per simulated second.

### 9.3 Simulation contract

The authoritative simulation:

- advances exactly 20 fixed ticks per simulated second;
- never reads wall-clock time;
- does not depend on rendering, audio, SDL events, or network arrival order;
- stores positions and velocities as signed Q20.12 fixed point with checked
  64-bit intermediates, truncation toward zero, and defined saturation before
  narrowing;
- assigns monotonic 32-bit entity IDs that are never reused within a match;
- updates entities in stable ID order;
- avoids pointer values, unordered-container iteration, locale, and platform
  math in state transitions;
- admits client-originated state changes only through validated commands;
- uses a pinned PCG32 algorithm for named random streams and keyed hashes for
  cell effects, with committed golden vectors;
- serializes explicit-width fields in little-endian order, never raw C++
  objects, padding, native enums, or container layout;
- has one canonical byte serialization shared by snapshots and hashing; and
- can run faster than real time without a window.

Floating point is allowed in rendering and other presentation systems. Render
state is interpolated from immutable simulation snapshots.

Every collection and priority queue has a total ordering. Canonical booleans
are one byte, enums are fixed-width integers, and length-prefixed collections
are serialized in their specified stable order. Dependencies, compiler
presets, cooked rules, and cooked map bytes are pinned and hashed; visual-only
assets have a separate hash.

Each tick uses this fixed system order:

1. Apply ordered match-control records and semantically validate/apply the
   tick's client commands. A terminal control record ends the tick immediately.
2. Advance power telegraphs and areas, applying Rain before ignition checks.
3. Update dry-season modifiers and the two-phase environment grid.
4. Update hunger, stockpiles, production queues, and Belief.
5. Run scheduled job assignment and the bounded path-request queue.
6. Resolve two-phase movement and occupancy.
7. Resolve gathering, deposits, construction, repair, and simultaneous combat
   damage.
8. Apply deaths, destruction, refunds, and completed construction.
9. Recompute dirty navigation and influence in stable order.
10. Recalculate World Heart presence, add score, and evaluate the Ascendancy
    target, time limit, sudden death, draw, or forfeit in that order.
11. Emit stable presentation events, advance the tick, and produce a scheduled
    digest or checkpoint.

### 9.4 Authoritative state

- Match tick, seed, rules version, and objective state
- Player resources, cooldowns, labor priorities, hand state, and match result
- Stable entity IDs and components required by gameplay
- Unit position, health, job, carried resource, hunger, and command state
- Structure position, health, construction state, connectivity, and production
- Navigation occupancy and reservations
- Influence sources and derived connected coverage
- Environment fuel, moisture, fire, and regrowth fields

Particles, animation phase, camera, cursor interpolation, sound instances,
screen shake, cosmetic debris, connection state, pending network buckets,
command history, checkpoint storage, and replay files are never part of the
hashed simulation state.

### 9.5 Commands

Initial command schema:

- `SetLaborPriorities`
- `SetVillagerProduction`
- `PlaceBuilding`
- `CancelBuilding`
- `TrainGuard`
- `SetGuardGroup`
- `MoveGuardGroup`
- `AttackMoveGuardGroup`
- `DefendGuardGroup`
- `PickupSubject`
- `DropSubject`
- `CastRain`
- `CastEmber`
- `Surrender`

The server can append `Forfeit`, `CancelHeldSubject`, and `NoContest`
match-control records. Connection pause/resume is recorded outside the tick
stream because it advances no authoritative time.

Every command includes protocol version, sequence number, target tick, type,
and a bounded type-specific payload. Player identity comes from the accepted
connection, never a trusted payload field. Receipt-time checks cover framing,
size, sequence, connection identity, and rate only. Semantic validation occurs
in canonical order on the target tick; an invalid command becomes a recorded
rejection event and cannot partially mutate state or spend resources.

### 9.6 Lockstep, hashes, and recovery

- Both clients use the same fixed four-tick command lead for the entire match.
- Clients send event-driven intents on a reliable ordered lane; they do not
  send heartbeat no-op commands.
- The server accepts an intent only while its target-tick bucket is open. A
  late intent for a closed tick is rejected and acknowledged without
  retargeting.
- At each 50 ms boundary, the server closes the next bucket, orders intents by
  `(player slot, client sequence, command index)`, executes it, and emits one
  canonical command bundle even when it is empty.
- Duplicate client sequences and duplicate bundles are idempotently ignored;
  bundle gaps stop client advancement until filled.
- Clients advance only when the bundle for the next tick is available.
- Local cursor, camera, previews, and acknowledgements respond immediately even
  though authoritative actions wait for their target tick.
- All three simulations hash the same canonical, uncompressed state bytes with
  XXH3-128 every 20 ticks. This detects divergence; it is not a security
  boundary.
- The server serializes a checkpoint every 100 ticks, caps canonical snapshots
  at 2 MiB, and retains at least 600 later tick bundles.
- A mismatching client pauses, receives the latest canonical checkpoint plus
  later command bundles in 32 KiB chunks on a lower-priority reliable lane,
  verifies the checkpoint, re-simulates, and resumes.
- A second mismatch after recovery ends the match as a technical failure and
  preserves diagnostics and the replay.

The handshake includes build ID, protocol version, rules-data hash, map hash,
and simulation schema version. Different values cannot enter the same match.
Server-side pause, resume, forfeit, surrender, and no-contest decisions become
ordered match-control records wherever they affect simulation outcome. A
wall-clock-only connection pause is replay metadata because no simulation tick
advances during it.

### 9.7 Replay

A replay contains:

- format and simulation versions;
- build and data hashes;
- map ID and match seed;
- match-local player slots and colors;
- the exact canonical tick-zero snapshot;
- every ordered command bundle, including empty ticks;
- ordered match-control records;
- expected per-second state hashes; and
- the final result and state hash.

Each replay chunk is length-delimited and checksummed. Playback runs through
the ordinary simulation path, verifies every scheduled digest, and rejects a
corrupt, truncated, or incompatible file. Presentation events use stable
`(tick, event index)` IDs so recovery or replay cannot duplicate audio and VFX.
A replay browser and seeking UI are not MVP features; a headless verifier and a
developer playback command are.

### 9.8 Navigation and AI

- One-meter deterministic navigation grid
- Integer movement and collision radii
- Deterministic eight-neighbor A* with fixed neighbor order and a total heap key
  of `(f, h, node index, insertion serial)`
- Ascending-entity-ID path request queue capped at eight new paths per tick
- Cached individual routes invalidated by changed building occupancy; no flow
  fields in the MVP
- Two-phase movement proposals resolved by ascending entity ID
- Job selection every ten ticks
- Work reservations to stop multiple villagers claiming the same final slot
- No navmesh generation, crowd middleware, ragdolls, or dynamic terrain

The first optimization target is avoiding needless searches, not introducing a
large generic ECS or job system.

### 9.9 Rendering

The custom renderer needs only:

- heightfield terrain;
- instanced low-poly units, trees, resources, and buildings;
- one directional light and simple shadows;
- influence and placement overlays;
- billboards or simple particles for rain, fire, smoke, and selection;
- a custom 2D UI and text layer; and
- GPU timestamps and debug labels.

The renderer consumes read-only presentation snapshots. It cannot be required
for tests, server operation, replay verification, or simulation progress.

### 9.10 Suggested source layout

```text
VillageSlop/
├── CMakeLists.txt
├── cmake/
├── data/
│   ├── rules/
│   └── maps/
├── docs/
├── shaders/
├── src/
│   ├── core/
│   ├── sim/
│   ├── net/
│   ├── render/
│   ├── platform/
│   ├── client/
│   └── server/
├── tests/
│   ├── determinism/
│   ├── simulation/
│   ├── networking/
│   └── replays/
└── tools/
```

Dependency direction:

```text
core <── sim <── net <── client/server
  └────────────── render <── client
platform ────────────────────┘
```

`sim` must not depend on `net`, `render`, or `platform`.

## 10. Content budget

The MVP is limited to:

| Category | Budget |
| --- | ---: |
| Maps | 1 fixed mirrored map |
| Players/factions | 2 mechanically identical players |
| Civilian types | 1 |
| Military types | 1 |
| Starting structures | 1 |
| Constructed structures | 4 |
| Divine hand interactions | Pickup and drop |
| Miracles | Rain and Ember |
| Disaster models | Fire plus one forecast dry season |
| Resources | Food, Wood, Belief |
| Victory objectives | World Heart Ascendancy |
| Camera modes | 1 |
| Multiplayer modes | 1 direct-address 1v1 mode |
| Player-facing screens | Title, connect/lobby, loading, match, result |
| HUDs | 1 match HUD plus local settings overlay |
| Replay surface | Headless verifier and developer playback command |
| Subject animation states | Idle, walk, work, worship, fight, flee |
| VFX families | Selection, influence, Rain, Ember, fire/smoke, objective |
| Audio | 1 ambient bed and at most 12 functional cues; no voice-over |
| Asset tooling | 1 deterministic offline cooker/packer; no editor |
| Diagnostics | Local structured log, replay, hashes, and timing capture |
| Online operations | 1 manually addressed referee server per match |
| Tutorial | Context prompts and one first-match checklist |
| Client platforms | 1 development client; cross-platform sim verifier only |

Anything that requires increasing this table must replace an existing item or
wait until the MVP is validated.

## 11. Explicitly outside the MVP

- A giant trainable creature or pet
- Terrain sculpting, erosion, or water simulation
- Rigid-body throwing and destruction
- Campaign, story, cutscenes, or voiced advisor
- More factions, cultures, ages, technologies, or tech tree
- Procedural maps or map editor
- Fog of war and scouting
- Diplomacy, trade, alliances, or matches larger than 1v1
- Matchmaking, accounts, ranking, progression, cosmetics, or monetization
- Spectators, host migration, tournaments, or anti-cheat guarantees
- NAT traversal, signaling, relay, room codes, or automatic server allocation
- Modding or general scripting
- Save-and-resume for live multiplayer matches
- Replay browser, editing, seeking, or compatibility across versions
- Certification of a second rendered client platform
- Mobile, console, or browser builds
- Production-quality asset pipeline
- Localization beyond keeping text data externalizable

These may be valuable later. They are excluded because they do not need to be
present to test the MVP promise.

## 12. Implementation gates

Each gate ends in an executable demonstration and automated evidence. A later
gate cannot redefine an earlier invariant.

### Gate 0 — Rules and data contract

Deliver:

- this agreed MVP rule set;
- normative precondition, mutation, rejection, refund, tie, and update-order
  tables for every command and system;
- versioned command, match-control, canonical-state, checkpoint, and replay
  schemas;
- fixed-point, RNG, entity-lifecycle, ordering, and canonical-codec contracts;
- dependency boundaries; and
- one deterministically cooked rules/map package with all tuning values;
- named reference hardware, stress scenario, network impairment envelope, and
  benchmark commands.

Pass when a rule-coverage checklist maps every client command and every match
ending to an exact state transition, failure result, serialized fields, and
test scenario without requiring a programmer to invent behavior.

### Gate 1 — Deterministic headless kernel

Deliver:

- CMake project and test executable;
- fixed-tick world;
- stable entity storage;
- command application;
- canonical serialization and hashing;
- deterministic movement on a grid; and
- replay writer/reader.

Pass when a committed corpus of at least 100 seeds by 20,000 ticks produces the
same scheduled hashes in repeated debug and optimized runs on macOS arm64 and
Windows x64. Fixed-point, PCG32, keyed-hash, and StateCodec golden vectors must
also match, and the provisional headless tick budget must pass.

### Gate 2 — Multiplayer before gameplay breadth

Deliver:

- headless referee server;
- two clients that connect and complete a handshake;
- ordered tick bundles;
- latency, jitter, loss, duplication, burst-loss, and reordering simulation;
- periodic state hashes;
- checkpoint recovery; and
- network/replay diagnostics.

Pass when two clients and the server complete the scripted match under 150 ms
RTT, 30 ms jitter, and 3% packet loss with zero spontaneous hash mismatch. A
separate one-byte fault test must detect the mismatch by the next digest and
recover within two seconds. The same build must also complete a session through
a public referee from two consumer networks.

### Gate 3 — First end-to-end networked match

Deliver:

- SDL window and input;
- custom GPU renderer;
- fixed heightfield, primitive subjects, Temples, Shrines, and World Heart;
- camera, selection, influence overlay, placement, and validated pickup/drop;
- connect/lobby, loading, accelerated objective, result screen, and developer
  replay playback; and
- the final client-command, referee, hash, result, and replay paths.

Pass when two remote humans connect, place Shrine chains, use the hand to move
preset subjects, contest the Heart, reach a result under accelerated scoring,
and verify the saved replay without console state mutation.

### Gate 4 — Autonomous village

Deliver:

- Food, Wood, Belief, storage, hunger, and population;
- job priorities and explainable job selection;
- the starting Temple and all four constructed structures;
- gathering, carrying, building, repair, worship, and villager production; and
- disconnected Shrine behavior.

Pass when a hands-off scripted village survives for ten simulated minutes and
a fixed scenario shows at least a 15% measured change in the chosen output when
priorities favor growth, expansion, or Belief.

### Gate 5 — Divine environment

Deliver:

- Rain, Ember, moisture, two-phase fire, wind, and regrowth;
- forecast dry season and paired natural ignition;
- readable telegraphs, forecast, fire-risk, and counterplay overlays; and
- deterministic golden fire-grid scenarios.

Pass when a player can intentionally start, prevent, contain, and exploit fire,
and both mirrored halves produce equivalent results when player slots and
authored strike cells are swapped.

### Gate 6 — Complete competitive match

Deliver:

- Guards and two-group commands;
- final World Heart scoring, time limit, sudden death, and draw;
- final HUD, local settings, onboarding checklist, result flow, disconnect
  timeout, surrender, and no-contest diagnostics; and
- the complete content budget with readable placeholder or MVP-quality assets.

Pass when two uncoached humans can play from connect screen to a recorded,
unambiguous result over the supported public-server envelope, and every ending
path has an automated replay fixture.

### Gate 7 — MVP hardening

Deliver:

- expanded cross-platform deterministic verification;
- per-gate performance budgets, profiling, packaging, and clean-checkout docs;
- input rebinding and color-independent ownership/danger indicators;
- retained failure replays and diagnostics; and
- balance revisions from external playtests.

Pass when every item in the Definition of Done is evidenced.

## 13. Verification strategy

### 13.1 Automated tests

- Fixed-point arithmetic edge cases and overflow policy
- Canonical serialization byte-for-byte fixtures
- Command validation, rejection, and idempotence
- Late, duplicate, malformed, and oversized command handling
- Entity creation/destruction order
- A* neighbor and tie-breaking cases
- Job selection and reservation conflicts
- Influence connectivity after Shrine loss
- Building placement rules
- Fire spread, rain extinguishing, dry-season, paired-ignition, and
  traversal-order-independence fixtures
- World Heart contest and all victory/tie paths
- Replay per-second and final-hash fixtures, including corruption rejection
- Snapshot restore followed by deterministic catch-up
- Client/server hashes under latency, jitter, loss, duplication, burst loss,
  and reordering
- Long-running soak simulation at population cap

### 13.2 Diagnostic evidence retained per match

- Build, protocol, rules, and map hashes
- Match seed and match-local player slots
- Ordered command log
- Per-second canonical state hashes
- Checkpoint metadata
- RTT, jitter, packet loss, and command-buffer depth
- Simulation and render frame timings
- First mismatch tick and a structured state difference when available

Diagnostic files contain no account, chat, or analytics identity. Server
connection addresses remain in short-lived operational logs, not replay or
gameplay telemetry. Gate 0 must define local paths, deletion controls, and a
default retention period before external tests begin.

### 13.3 Playtest questions

After each match, collect:

- Did both players understand why the World Heart was or was not scoring?
- Could each player explain how Food, Wood, Belief, and influence connect?
- Did villagers appear purposeful rather than broken or arbitrary?
- Did using the hand feel powerful without replacing village logistics?
- Was an incoming Ember readable, and did Rain feel like a real counter?
- Did either player feel eliminated long before the match ended?
- What action did each player want to take but could not express?

Instrumentation should record match length, first Shrine timing, first World
Heart contact, control changes, labor allocations, population curves, Belief
income and spending, Guard counts, miracle use, burned cells, structure losses,
and final Ascendancy.

## 14. Definition of Done

The MVP is complete only when all of the following are true:

### Complete experience

- An operator can start the documented public referee command, and two players
  on separate consumer networks can connect by address.
- Both enter the same match with validated content and protocol versions.
- Both can complete the full economy, expansion, miracle, combat, and objective
  loops.
- Every match ends through Ascendancy, time-limit comparison, sudden death,
  draw, surrender, disconnect forfeit, or a labeled technical no-contest.
- The result screen identifies the winner and decisive score, or the exact
  draw/no-contest reason.
- The saved replay reaches the recorded final hash and result.

### Multiplayer and determinism

- The server is the command-order and validation authority.
- Invalid or unaffordable actions cannot mutate authoritative state.
- Automated network tests cover latency, jitter, loss, duplication, burst loss,
  reordering, closed-tick commands, and corrupt payloads.
- An interrupted existing session resumes inside the 30-second grace period;
  an expired grace period produces the recorded forfeit result.
- Fifty consecutive full scripted matches finish with matching server and
  client hashes at every scheduled digest and require no spontaneous recovery.
- A deliberately corrupted client is detected by the next digest and rejoins
  the canonical timeline within two seconds through checkpoint recovery.
- The same replay reaches identical scheduled hashes on macOS arm64 and Windows
  x64.

### Performance and stability

Gate 0 records the exact reference CPU, GPU, OS, build preset, and benchmark
commands. The shared stress fixture uses 40 subjects, six Shrines, and all
other structure caps per player, with 25% of environment cells actively
burning.

- On that fixture, simulation ticks are below 5 ms p95, 10 ms p99, and 20 ms
  maximum excluding checkpoint ticks; checkpoint ticks remain below 30 ms.
- The rendered stress fixture holds at least 60 frames per second at 1080p,
  measured as no more than 16.7 ms p95 over ten minutes.
- Steady-state match traffic, excluding deliberate snapshot recovery, remains
  below 32 KiB/s per player p95; a canonical snapshot remains at or below
  2 MiB.
- A four-hour headless soak is assertion-, sanitizer-, overflow-, and
  divergence-free, with resident memory growing less than 1 MiB per hour after
  a 30-minute warm-up.

### Usability and game validation

- At least nine of twelve first-time participants identify, within five
  seconds and without debug overlays, whether placement is valid, who controls
  the Heart, where Ember will land, and when the dry season begins.
- Color is not the sole signal for player ownership or fire danger.
- Controls can be rebound and camera sensitivity can be changed.
- Twelve first-time participants complete six paired matches without spoken
  coaching, followed by at least four side-swapped repeat matches.
- At least eight of those ten measured matches finish from 12:00 through 20:00;
  any tied match must finish or draw by 21:00.
- At least nine of twelve first-time participants can explain the core
  economy–belief–influence–objective loop after one match.
- Each of four documented opening scripts has at least one response that wins
  or reaches a tied 20:00 score in a side-swapped deterministic scenario.

### Scope integrity

- The shipped content does not exceed the MVP budget without an explicit
  replacement decision.
- No excluded feature is required for the complete match.
- A clean checkout has documented build, test, host, join, and replay commands.

## 15. Completion evidence

Completion is audited from named artifacts, not memory or an informal
playthrough:

| Requirement | Required evidence |
| --- | --- |
| Rules are closed | Versioned cooked rules/map package and rule-coverage report |
| Determinism | Cross-platform golden-vector and replay-hash CI reports |
| Network correctness | Impairment matrix, fault-injection recovery report, and 50-match hash report |
| Real online play | Public-referee logs and replays from two consumer networks |
| Complete experience | Six first-time and four repeat-match replays with result records |
| Disaster counterplay | Mirrored fire golden fixtures and playtest event logs |
| Performance | Named-hardware simulation, render, memory, snapshot, and bandwidth reports |
| Usability | Anonymized first-time task and comprehension results |
| Reproducible delivery | Clean-checkout CI logs, package checksums, and documented commands |

Gate 7 cannot pass when an artifact is missing, stale for the candidate build,
or covers a narrower scenario than its requirement.

## 16. Primary risks and responses

| Risk | Early warning | Response |
| --- | --- | --- |
| Cross-platform desync | First hash mismatch occurs only on another compiler or CPU | Run cross-platform replay fixtures from Gate 1; prohibit floating point and unstable iteration in simulation |
| Autonomous villagers feel unresponsive | Players repeatedly fight priorities or use the hand as a workaround | Expose job reasons, reservations, and urgency; simplify job selection before adding behaviors |
| Lockstep feels laggy | Placement and powers feel delayed at ordinary RTT | Immediate local previews, visible pending state, fixed equal command lead, and no speculative authoritative mutation |
| Fire decides matches arbitrarily | One ignition causes unrecoverable damage without a readable mistake | Telegraph Ember, forecast dryness, provide firebreaks and Rain, cap spread, and log causal cells |
| Shrine loss creates a hopeless snowball | Losing one contest removes all safe actions | Preserve base influence, make chains legible, keep rebuilding affordable, and measure recovery time before adding opaque comeback bonuses |
| Custom engine consumes the project | Rendering or tooling gates expand without proving a match | Enforce the content budget and require every gate to end in a playable or testable vertical result |
| RTS micro overwhelms god fantasy | Best players individually manipulate every subject | One-object hand limit, cooldown, civilian indirect control, and group-only Guard commands |
| Match stalls around the center | Neither player risks contesting | Accumulated Ascendancy, hard time limit, forecast dry season, and three broad approaches |

## 17. Decision log

These decisions define the MVP scope baseline:

1. Multiplayer uses deterministic lockstep with a simulating referee server.
2. The MVP has complete information and no fog of war.
3. The authoritative simulation uses fixed-point/integer state at 20 Hz.
4. The World Heart and accumulated Ascendancy are the only primary victory
   system.
5. Villagers are indirectly controlled; Guards receive group commands.
6. The hand performs validated pickup/drop without rigid-body physics.
7. Rain, fire, and one dry season are the complete MVP disaster model.
8. The map is fixed and mirrored.
9. SDL3 and GameNetworkingSockets are boundary libraries; there is no game
   engine.
10. The giant creature, terrain deformation, procedural maps, and metagame are
    postponed until the MVP promise is proven.
11. The online MVP uses a manually addressed public referee server; connection
    discovery and NAT traversal are postponed.
12. Both players use the same four-tick command lead for the full match.
13. The game must end by 21:00 of simulated time unless a connection pause
    stops simulation time.

## 18. Reference documentation

- [SDL3 overview](https://wiki.libsdl.org/SDL3/FrontPage)
- [SDL3 GPU API](https://wiki.libsdl.org/SDL3/CategoryGPU)
- [GameNetworkingSockets](https://github.com/ValveSoftware/GameNetworkingSockets)
- [xxHash](https://github.com/Cyan4973/xxHash)
- [Khronos Vulkan guide](https://docs.vulkan.org/guide/latest/what_is_vulkan.html)
