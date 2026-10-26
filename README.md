# tb3sim

Pure-Python 2D TurtleBot3 Burger sim (Gymnasium): differential drive, 360-sample LDS-01 LiDAR
binned to 24 beams, circular/box obstacles, random goals, 5 discrete actions. No ROS or Gazebo needed.
Extracted from [vt-turtlebot/tb3_dqn](https://github.com/vt-turtlebot/tb3_dqn).

```bash
pip install -e .
```
```python
from tb3sim import make_fast_env
env = make_fast_env()              # settings live in src/config.yaml
obs, info = env.reset(seed=0)
obs, r, terminated, truncated, info = env.step(env.action_space.sample())
```

Limit: `config.yaml` is read from `src/`, so use an editable install (or pass `config_path=`).
