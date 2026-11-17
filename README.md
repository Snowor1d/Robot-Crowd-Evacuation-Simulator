# ADDS Simulator

A robot-guided crowd evacuation simulator. Evacuees move through an indoor or
outdoor map with obstacles and exits; a guide robot moves among them, and
evacuees who notice it may follow it to an exit. The simulator measures how
long the evacuation takes.

## Features

- **Maps**: obstacles and exits as polygons, loaded from `maps/map_<n>.json`,
  from built-in layouts, or generated at random (`map_num=0`). Optional random
  rotation / reflection of a map.
- **Navigation mesh**: the free space is triangulated (constrained Delaunay,
  `triangle`), with shortest routes between triangles (Floyd-Warshall). It
  gives walking distances and the next waypoint toward a goal.
- **Visibility**: what an agent can see within its vision radius, past
  obstacles, precomputed per map region.
- **Evacuees**: social-force motion (driving force, exponential repulsion,
  non-penetrating contact with damping and friction, wall and border
  repulsion, swept movement against tunnelling). Each evacuee chooses an exit
  it can see by a score of walking distance, crowding and exit width, follows
  the robot when it is close, follows other people, or explores, and keeps a
  decision for a random time before reconsidering it.
- **Guide robot**: moves by a velocity command, with wall repulsion. Controlled
  by your own code (`robot.receive_action`), by the built-in rule "walk to
  the nearest exit", by an operator (joystick), or left out.
- **Rendering**: vector-style frames (matplotlib) with trails, heading
  arrows, navigation mesh and distance overlays; live view in pygame, video
  and PNG output.

## Installation

```
pip install -r requirements.txt
```

## Layout

```
adds_sim/              the simulator package
  model.py             EvacuationModel: map, navigation mesh, agents, step loop
  agent.py             CrowdAgent (evacuee) and RobotAgent
  space.py             continuous space with a grid index for neighbour queries
  visibility_atlas.py  precomputed visibility polygons
  map_augmentation.py  rotations / reflections of a map
  random_map.py        random map generator (map_num=0, map editor)
  renderer.py          ContinuousRenderer (frames as RGB arrays)
  core.py              minimal agent-based modelling core (Agent, Model, scheduler)
  config.py            simulation parameters
maps/                  map files (map_<n>.json)
assets/                robot image used by the renderer
run_simulator.py       interactive simulator (GUI)
manual_control.py      drive the robot with a joystick and record the run
map_editor.py          draw and edit maps
make_random_map.py     simple random map generator (command line)
```

## Quick start (Python)

```python
from adds_sim import EvacuationModel

env = EvacuationModel(number_agents=30, width=100, height=100, map_num=1500,
                      robot_control="go_to_exit")
while env.alived_agents() > 0 and env.step_count < 3000:
    env.step()
print("evacuation time:", env.step_count * 0.5, "s")
```

One step is `AGENT_TIME_STEP` = 0.5 s of simulated time.

`robot_control`:

| value | the robot |
| --- | --- |
| `"external"` | follows the velocity command `(vx, vy)` in m/s given with `env.robot.receive_action([vx, vy])`; the command is kept until the next one |
| `"go_to_exit"` | walks to the nearest exit (built-in rule) |
| `"none"` | no robot |

A controller of your own sets a command and then steps the model:

```python
env = EvacuationModel(30, 100, 100, map_num=1502, robot_control="external")
for _ in range(3000):
    rx, ry = env.robot.xy
    env.robot.receive_action([1.0, 0.0])     # your decision here
    env.step()
    if env.alived_agents() == 0:
        break
```

Useful state: `env.crowds` (evacuees with `.xy`, `.vel`, `.dead`, `.type`:
0 following the robot, 1 own way, 2 following others), `env.robot`,
`env.exit_list`, `env.obstacles`, `env.alived_agents()`,
`env.evacuated_agents()`, `env.evacuated_agents_with_robot()`,
`env.calculate_all_agents_life_time()`, `env.path_distance(p, q)` (walking
distance).

Drawing a frame:

```python
from adds_sim import ContinuousRenderer
renderer = ContinuousRenderer(world_size=(100, 100), robot_style="image")
rgb = renderer.draw(env, step=env.step_count)      # (H, W, 3) uint8
```

## Programs

| command | what it does |
| --- | --- |
| `python3 run_simulator.py` | live simulation: pause / step / reset, speed, map number, number of evacuees, robot controller. Keys: Space, Right, Up/Down, Esc |
| `python3 manual_control.py` | drive the robot with the on-screen joystick or the arrow keys; writes metrics, the number of people inside per step and a video to `results/manual_control/` |
| `python3 map_editor.py` | draw obstacles and exits, generate random maps, trace a background image, save `maps/map_<n>.json` (asks for the map size in the terminal) |
| `python3 make_random_map.py --seed 42 --out maps/map_9000.json` | write a simple random map |

## Map format

```json
{
  "width": 100,
  "height": 100,
  "obstacles": [ [[x, y], [x, y], ...], ... ],
  "exits":     [ [[x, y], [x, y], ...], ... ]
}
```

Coordinates are in metres, with the origin at the bottom-left corner.
`maps/` holds 428 maps (numbers 1 to 4003). Map numbers without a file are
built-in layouts defined in `EvacuationModel.extract_map`, and `map_num=0`
generates a random map.

## Configuration

`adds_sim/config.py` holds the parameters: the default map and the pool of
random maps, random map transforms, the number of evacuees, body radii,
vision ranges, walking speed, the robot's maximum speed, the time step and
the weights of the exit score.
