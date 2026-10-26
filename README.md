# tb3sim

A pure-Python 2D simulator of the **TurtleBot3 Burger** as a [Gymnasium](https://gymnasium.farama.org/)
environment. The robot drives to a random goal across a square arena with obstacles, using only a
24-bin LiDAR and its goal-relative pose. It needs only NumPy, Gymnasium and PyYAML. No ROS and no Gazebo.

It is the "fast tier" of [vt-turtlebot/tb3_dqn](https://github.com/vt-turtlebot/tb3_dqn), where it is
used for 200k-step DQN training. The same task also exists as a Gazebo (ROS 2 Jazzy) env, so a policy
trained here can be tested in Gazebo. The Gazebo env (`envs/env_gz.py`) is included but is only
imported if you ask for it, and it needs ROS 2.

## Install

```bash
git clone https://github.com/s2anjay/tb3sim.git
cd tb3sim
pip install -e .          # editable install is required, see "Limits"
```

Python 3.10+.

## Use

```python
from tb3sim import make_fast_env

env = make_fast_env()                                  # settings come from src/config.yaml
obs, info = env.reset(seed=0)

done = False
while not done:
    action = env.action_space.sample()                 # replace with your policy
    obs, reward, terminated, truncated, info = env.step(action)
    done = terminated or truncated

print(info["outcome"])                                 # "goal", "collision" or "timeout"
```

`make_fast_env(config_path=None, overrides=None)`:
- `config_path`: use a different YAML file instead of `src/config.yaml`.
- `overrides`: list of `"section.key=value"` strings, e.g. `["task.max_steps=300", "reward.progress_weight=10"]`.

To replay a specific layout, pass it to `env.reset(options={"layout": layout})`. Layouts are
`envs.layout.Layout` objects. By default `reset` samples a new random one from the seed.

## The environment

**Observation**: `float32` vector of length 28.

| Index | Value | Range |
|-------|-------|-------|
| 0-23 | LiDAR bins (min over a 15 degree sector), clipped at 3.5 m and divided by 3.5 | 0..1 |
| 24 | distance to goal, divided by 5.66 (the arena diagonal) | about 0..1 |
| 25 | heading angle to goal, wrapped to [-pi, pi] radians | -pi..pi |
| 26 | last linear velocity, divided by 0.22 | 0..1 |
| 27 | last angular velocity, divided by 2.84 | -1..1 |

**Actions**: `Discrete(5)` at a constant forward speed of 0.15 m/s with angular velocity
`[1.5, 0.75, 0.0, -0.75, -1.5]` rad/s, i.e. hard left, left, forward, right, hard right.

**Episode ends** (`terminated` or `truncated`):
- goal reached: distance to goal < 0.2 m (`terminated`)
- collision: minimum LiDAR reading < 0.13 m (`terminated`; wins over goal if both happen)
- timeout: 500 steps (`truncated`)

**Reward**: `50 * (previous_distance - distance) - 0.05` every step, plus +100 on goal and -100 on collision.
The progress term is potential-based, so circling nets zero.

**`info`** from `step`: `outcome`, `success`, `dist`, `min_range`, `v`, `w`, `dw` (|change in w|, a
smoothness measure) and `pose` (x, y, yaw). `reset` returns `pose`, `goal` and `min_range`.

**Simulation**: control rate 10 Hz, 5 physics sub-steps per control step, acceleration limit 1 m/s^2,
a 360-sample LDS-01 scan (0.12-3.5 m, Gaussian noise 0.01) binned to 24. The arena is a 4 m square
with 3-6 random cylinders and boxes. Start and goal positions are sampled with clearance from walls
and obstacles.

Optional actuation realism (all default to 0, meaning instant actuation): `run.action_noise_std`,
`run.act_delay_s`, `run.act_delay_jitter_s`, `run.act_lag_tau_s`. These are the knobs used in the
parent project to study the sim-to-Gazebo gap.

## Configuration

Every tunable value is in [`src/config.yaml`](src/config.yaml): robot limits, LiDAR, arena and
obstacle sizes, task thresholds, reward weights, and the action set. The `dqn`, `eval` and `gz`
sections belong to the parent project's training code and are ignored by the env itself.

## Layout

```
pyproject.toml
src/
  config.yaml        all settings
  utils.py           config loading and overrides
  tb3sim/            make_fast_env()
  envs/
    env_fast.py      FastEnv: the simulator
    common.py        base env: observation, reward, termination
    layout.py        random arena layouts
    env_gz.py        Gazebo env (needs ROS 2, not used here)
    gz_world.py      Gazebo world generator
```

## Limits

- **Editable install only.** `config.yaml` is read from `src/`, so a built wheel installed elsewhere
  will not find it unless you pass `config_path=`.
- **Generic top-level names.** The package installs top-level modules called `envs` and `utils`,
  which can clash with other projects.
- **It is an approximation.** The motion model and ray casting are re-implemented in NumPy, not
  taken from Gazebo, so a policy trained here can behave differently on the real robot or in Gazebo.
- **No tests yet.** It has been smoke-tested by resetting and stepping with random actions only.
