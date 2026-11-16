# Setup and reproducibility

This guide separates source preparation from runtime prerequisites. The portfolio review performed static checks and LFS payload verification; it did not build ROS 2 packages or execute CARLA/Autoware.

## Compatibility baseline

| Component | Repository evidence | Verification boundary |
|---|---|---|
| CARLA | Python client pinned to `0.9.16` in `docker/requirements.txt`; upstream setup targets the same server | Use a native-ROS-2-enabled server with compatible client and content. Stock packaged-server asset loading is not established here. |
| ROS 2 | `ros:humble-ros-base-jammy` Docker base | Humble / Ubuntu 22.04 is the configured bridge environment. |
| Autoware Universe | External `autoware_launch e2e_simulator.launch.xml` workflow; upstream README recommends 1.7.1 | No complete Autoware commit/manifest or build record is stored here. Verify launch arguments, packages, plugins and messages against your checkout. |
| Message packages | `docker/ros2.repos`: `autoware_msgs 1.4.0`, Tier IV messages `v0.41.0`, ADAPI commit, internal messages `1.8.0` | Pins describe the bridge overlay, not a tested full-stack release combination. |
| Evaluation dependencies | `docker/carla.repos` selects two TUM branches | Branch tips can move; record exact commits and any local patches before a run. |

## Obtain the three repositories

Install Git LFS, vcstool, colcon, rosdep, build tools and ROS 2 Humble in the intended build environment. Git LFS must be installed before retrieving the mesh/model payloads.

```bash
export FRMAX_WORKSPACE="$HOME/frmax-simulation"
mkdir -p "$FRMAX_WORKSPACE"
cd "$FRMAX_WORKSPACE"
git lfs install
git clone https://github.com/gokcerSnmzck/autoware_carla_leaderboard_custom.git
git clone https://github.com/gokcerSnmzck/carla_yuhesen_fr_max.git
git clone https://github.com/gokcerSnmzck/carla_fr_maxv10.git
git -C carla_yuhesen_fr_max lfs pull
git -C carla_yuhesen_fr_max lfs fsck
git -C carla_fr_maxv10 lfs pull
git -C carla_fr_maxv10 lfs fsck
export FRMAX_MAIN="$FRMAX_WORKSPACE/autoware_carla_leaderboard_custom"
```

Keep the repositories separate. Their current layout comprises one bridge/evaluation workspace, one set of Autoware packages, and one CARLA content overlay.

## CARLA and the vehicle model

Follow the [upstream CARLA/AWCL installation instructions](https://github.com/TUMFTM/autoware_carla_leaderboard#run) for the native ROS 2 server build and [source dependency](https://github.com/TUMFTM/autoware_carla_leaderboard). The inherited [design notes](../doc/Design.md) describe a modified CARLA clock namespace: the bridge subscribes to `/carla/clock`, not the server's ordinary `/clock`. Verify the installed build's clock and TF topics before proceeding.

Set `CARLA_ROOT` to that installation. Back up its content before merging assets: this overlay includes shared `VehicleFactory`, base pawn, tire and wheel files, not just a self-contained robot folder. Read the [model installation guide](https://github.com/gokcerSnmzck/carla_fr_maxv10#installation) before copying. Do not overwrite an unrelated simulator installation.

```bash
export CARLA_ROOT=/path/to/CARLA_0.9.16
# After backing up the destination and checking build compatibility:
cp -a "$FRMAX_WORKSPACE/carla_fr_maxv10/CarlaUE4/." "$CARLA_ROOT/CarlaUE4/"
```

The 34 Unreal payloads are available remotely and were hash-verified during review. `VehicleFactory.uexp` contains `fr-maxv10`; this supports the configured model label, but the full runtime blueprint ID must come from the server:

```python
import carla
client = carla.Client("localhost", 2000)
client.set_timeout(10.0)
world = client.get_world()
print("server:", client.get_server_version(), "client:", client.get_client_version())
print([bp.id for bp in world.get_blueprint_library().filter("*fr-max*")])
```

Run the query after launching the server. An empty result means content loading/registration is incomplete. These cooked asset files alone do not prove compatibility with every CARLA package or Unreal build.

## Bridge workspace

From a ROS 2 Humble environment, install Python requirements into an environment that can also import `rclpy`. The system Python in the configured Jammy image is Python 3.10; do not mix incompatible Python interpreters and ROS installations.

```bash
cd "$FRMAX_MAIN"
source /opt/ros/humble/setup.bash
mkdir -p src/external
vcs import --recursive src/external < docker/carla.repos
vcs import --recursive src/external < docker/ros2.repos
python3 -m pip install -r docker/requirements.txt
# Initialize rosdep once if this environment has not already done so:
# sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -r -y --rosdistro humble
colcon build --symlink-install --cmake-args -DCMAKE_BUILD_TYPE=Release
source install/setup.bash
source carla_envs.sh
```

`carla_envs.sh` must be **sourced**, so the evaluator receives the agent and external-package `PYTHONPATH`. Run evaluation from the repository root because YAML paths are relative to the working directory. `agent-config` now points to `config/config.yaml`; both current agent `setup()` methods accept but do not parse that argument. Runtime agent selection is under `evaluation.agent_setup.agent`.

For an isolated bridge environment, the repository also supplies:

```bash
cd "$FRMAX_MAIN"
docker build -f docker/Dockerfile -t frmax-awcl:local .
docker run --rm -it --network host \
  -v "$FRMAX_MAIN:/carla" -w /carla frmax-awcl:local bash
```

Build the overlay inside that container using the colcon command above. The Dockerfile installs dependencies but does not run colcon; the bind mount replaces image content at `/carla`. The entrypoint sources `/carla/install/setup.bash` only when it exists. Autoware and the CARLA server need their own environments; use compatible ROS domain and DDS settings across processes. Docker availability alone is not evidence that a GPU simulation environment is ready.

## Autoware platform packages

Prepare a complete Autoware Universe workspace using the selected release's own dependency instructions. The inherited README recommends 1.7.1; treat it as a candidate baseline, not a verified FR-max compatibility guarantee. Set `AUTOWARE_WS` to the workspace root, which must contain `src`, and `MAP_PATH` to the matching Autoware map directory (not the CARLA Unreal map).

```bash
export AUTOWARE_WS=/path/to/autoware
export MAP_PATH=/path/to/Town01
mkdir -p "$AUTOWARE_WS/src/vehicle"
ln -s "$FRMAX_WORKSPACE/carla_yuhesen_fr_max" \
  "$AUTOWARE_WS/src/vehicle/carla_yuhesen_fr_max"
cd "$AUTOWARE_WS"
source /opt/ros/humble/setup.bash
colcon list | rg yuhesen_fr_max
rosdep install --from-paths src --ignore-src -r -y --rosdistro humble
colcon build --symlink-install --packages-up-to \
  yuhesen_fr_max_vehicle_launch yuhesen_fr_max_vehicle_description \
  yuhesen_fr_max_sensor_kit_launch yuhesen_fr_max_sensor_kit_description \
  yuhesen_fr_max_common_sensor_launch
source install/setup.bash
ros2 pkg prefix yuhesen_fr_max_vehicle_description
ros2 pkg prefix yuhesen_fr_max_sensor_kit_launch
```

Create the symlink once, or clone the supporting repository directly into that source location instead. Include all five packages; selecting only the two launch packages omits description/common packages. `--packages-up-to` includes declared dependencies, but cannot compensate for missing runtime dependency declarations; verify the external packages below. A normal full Autoware build must already supply `autoware_launch` and the autonomy stack.

Map assets are provided by [upstream AWCL](https://github.com/TUMFTM/autoware_carla_leaderboard/releases/tag/v1.0.0-maps). Match Town01 routes to a Town01 map and the Town10HD route to its corresponding Autoware map.

## Execution prerequisites

Before claiming FR-max/cyclist execution, resolve these source-confirmed interface conditions in a separate behavior-change review:

1. **Ego spawning:** inspected TUM Leaderboard `029b65d42d26d4e029c15b97335b9d0f918e5766` hard-codes `vehicle.audi.etron`. It does not consume `ego_vehicle_model`. Adapt it to select the confirmed runtime FR-max blueprint, propagate configuration deliberately, and fail clearly if the blueprint is absent.
2. **Cyclist parameters:** inspected Scenario Runner `0ec3b37897ac8c93af74e8dfeec5427a131f3acc` hard-codes a Nissan lead actor, 25 m offset and 10 m/s. Adapt `FollowLeadingVehicle` to consume and validate the authored model/distance/velocity parameters, retaining baseline defaults for unrelated routes.
3. **Sensor contract:** FR-max E2E specs request 32/16 channels while the C++ launch uses 128/64; Xacro uses VLS-128 macros. Verify the intended model, point layout and channel encoding before changing any of them. The privileged sensor specs omit `ros_name`, which the declared Leaderboard sensor wrapper requires; that mode also publishes only the front cloud.
4. **Frames and topics:** bridge inputs assume `/carla/ego_vehicle/...`; spawned role is `hero`. Verify actual DDS names. Compare CARLA actor origin, rear-axle `base_link`, +0.425 m sensor-kit offset, camera optical frame and sensor extrinsics. Agent/C++ camera is 800×400; Xacro is 800×600. No calibration is changed by this portfolio update.
5. **Launch composition:** `pointcloud_preprocessor.launch.py` declares `container_name` but reads `pointcloud_container_name`; it also reads `use_concat_filter` without declaring it locally. Verify parent scope and container namespace before changing behavior. `carla_lidar_node_container.launch.py` declares `container_name` twice. Validate full launch expansion in the chosen Autoware environment.
6. **External packages:** launch files reference `autoware_pointcloud_preprocessor`, `autoware_gnss_poser`, `autoware_imu_corrector`, `autoware_vehicle_velocity_converter`, VLS/camera/IMU descriptions, while some manifest entries retain older unprefixed names or omit those dependencies. Compare names and plugin exports against the selected Autoware checkout before correcting manifests. Hardware driver and GNSS conversion paths coexist with simulated feedback; verify `launch_driver` and effective publishers to avoid conflicting sources.

These checks explain why the commands below are a launch procedure after environment preparation, not a claim that fresh clones produce a working FR-max experiment. No spawning, control, sensor calibration or physics changes are bundled into the portfolio update.

## Launch sequence

Use separate terminals and export the path variables above in each relevant environment. Source the matching ROS/workspace setup in every terminal.

1. Start the compatible CARLA server:

   ```bash
   cd "$CARLA_ROOT"
   ./CarlaUE4.sh --ros2
   ```

2. Start the bridge in the bridge environment:

   ```bash
   cd "$FRMAX_MAIN"
   source /opt/ros/humble/setup.bash
   source install/setup.bash
   ros2 launch autoware_carla_cpp_bridge aw_carla_cpp_bridge.launch.py
   ```

3. Start Autoware in its own prepared environment:

   ```bash
   cd "$AUTOWARE_WS"
   source install/setup.bash
   ros2 launch autoware_launch e2e_simulator.launch.xml \
     vehicle_model:=yuhesen_fr_max_vehicle \
     sensor_model:=yuhesen_fr_max_sensor_kit map_path:="$MAP_PATH"
   ```

   Verify this launch file and its arguments with `ros2 launch autoware_launch e2e_simulator.launch.xml --show-args` first. The vehicle/sensor selectors correspond to package prefixes, not to the outer repository directory names.

4. After Autoware is ready, run the selected route from the main repository root:

   ```bash
   cd "$FRMAX_MAIN"
   source install/setup.bash
   source carla_envs.sh
   python3 src/external/leaderboard/leaderboard/leaderboard_evaluator.py \
     --conf_file_path=config/config.yaml
   ```

Change `evaluation.simulation_setup.routes` to select another [scenario](SIMULATION_SCENARIOS.md), and match its map. Evaluation output paths are `simulation_results.json` and `live_results.txt`; recording is disabled by default. Preserve outputs as evidence and verify actual ego and lead actor IDs before interpreting results.
