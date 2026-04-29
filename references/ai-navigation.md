# AI and Navigation

## Objective
Provide a healthy foundation for pathfinding, navigation, and simple behavior in Three.js games, without jumping straight to industrial solutions when the game does not need them.

## Main rule
**Before adding a navigation engine, prove that “dumb” movement is not enough.**
Many games can be solved with steering + raycast + waypoints, without a nav mesh.

## Three navigation levels
Scale from lower to higher cost/complexity:

### Level 0 — Direct movement + steering
- move toward a target with limited acceleration.
- basic separation between agents.
- obstacle avoidance with raycasts forward/sides.
- enough for open scenarios with few obstacles and few agents.

### Level 1 — Waypoint graph
- manually placed or generated nodes, with visible edges.
- A* over the graph.
- enough for levels with limited routes (patrols, rounds, key points).
- cheap, controllable, easy to debug visually.

### Level 2 — Nav mesh
- walkable surface generated from level geometry.
- A* over polygons, string-pulling for natural paths.
- necessary for open worlds or levels with lots of irregularity.
- typical web stack: `recast-navigation` (Recast/Detour port) or similar options. They are external addons: mark them as such.

## Choosing a level
Questions:
- how many simultaneous agents? (>10–20 starts asking for more than raycast)
- complex geometry or simple world?
- static or dynamic levels?
- tight paths (bridges, narrow corridors) or wide spaces?
- are levels reused or different every run?

Answer before choosing an engine.

## Integration with physics
Navigation is not physics:
- the pathfinder decides *where* to go.
- physics decides *how* the body moves in the real world (collisions, gravity).
- an agent combines: pathfinder → waypoints → locomotion/controller → physics.

Do not put pathfinding logic inside a rigid body.

## Path updates
- recalculate the path only when the target or world changes, not every frame.
- re-path on demand if the agent stays blocked for N frames.
- if there are many agents, distribute re-paths across several frames (tick budget).

## Steering and follow-path
- the path is a list of points, not a rail.
- use look-ahead: the agent aims at a point some distance along the path, not strictly at the next waypoint.
- use string-pulling to smooth paths over a nav mesh.
- arrival tolerance (`arriveRadius`) on every waypoint.

## Local agents and separation
With several agents:
- basic distance-based separation between pairs (with a spatial grid to avoid O(n²)).
- prioritization: the agent with less progress yields.
- never push with physics unless it is part of the design.

## Simple behavior
Before behavior trees or utility AI:
- one finite state machine (FSM) per enemy: `idle`, `patrol`, `chase`, `attack`, `flee`.
- transitions by conditions (distance, visibility, health).
- enough for prototypes and many small games.

If behavior grows:
- behavior tree (external addon or simple custom implementation).
- utility AI if there are many actions with changing priorities.

Never start with a behavior tree “because it sounds professional”.

## Perception
- sight: raycast from agent to target; FOV cone with direction `dot`.
- hearing: events emitted by player actions with a radius; subscribed agents filter by distance and obstacles.
- short memory: the agent remembers the last known position for X seconds.

Explicit perception modeling avoids enemies that always know everything.

## Compute distribution
- do not run AI for every agent every frame. Tick staggered: a subset each frame.
- scale by distance/importance: distant agents think less and move with lower fidelity.
- if there are many, use AI LOD: far away, patrol only; nearby, full FSM.

## Debug
- visualize paths with lines.
- draw vision cone and hearing radius.
- color the agent by state.
- overlay AI cost per frame.

## Dynamic world
- obstacles that appear/disappear invalidate paths.
- with nav mesh: support for tiles or dynamic patches (most libs expose this).
- with waypoints: mark edges temporarily blocked.

## Mobile
- large nav meshes consume memory and CPU: limit the area.
- fewer active agents, more aggressive LOD.
- avoid massive re-paths at the same time.

## Anti-patterns
- adding a Recast/Detour port for 5 enemies on a plane
- running A* every frame “just in case”
- deciding behavior with ifs scattered through entities
- physics solving navigation (“I push the agent until it arrives”)
- enemies seeing through walls because there is no visibility raycast
- one monster AI tick with every agent every frame
- nav mesh regenerated at runtime without need

## Strong recommendation
Healthy default flow:
1. start with steering + raycast.
2. if that fails, add waypoints + A*.
3. if the world requires it, then use a nav mesh.
4. FSM as default behavior; behavior tree only when the number of states justifies it.

## Related references
- `character-locomotion.md`
- `physics.md`
- `world-generation.md`
- `debugging.md`
- `mobile-performance.md`
