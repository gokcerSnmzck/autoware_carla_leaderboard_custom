# Technical contributions and provenance

## Evidence basis

The local main-repository history was compared with the public TUM repository fetched on 2026-11-16. Commit `0ecb224` exists upstream and is the parent of Gökcer Sönmezocak's implementation commit `d3dff21`. The inspected upstream HEAD, `afc38611742bb56879d6b315f1afae542e7565c7`, differs from that base only in README content. This provides a concrete source baseline, rather than inferring ownership from file names.

```bash
git show --stat d3dff21
git diff 0ecb224 d3dff21 -- src config resources/routes
```

## Main repository ownership

| Component | Inherited work | Gökcer's recorded contribution |
|---|---|---|
| [Control converter](../src/tum_agents/autoware_agent/aw_converter.py) | Steering interpolation, coordinate conversion, acceleration-feedback structure, anti-windup, object conversion | Vehicle-model constructor argument; ±25° FR-max steering table; FR-max gains and actuator limits; parameterized LiDAR channel count and explicit `uint16` channel IDs. |
| [E2E agent](../src/tum_agents/autoware_agent/aw_e2e.py) | ROS topics, routing/engagement, clock/control queue, pose/status/IMU publication, baseline Audi sensors | Actor `type_id` propagation, FR-max sensor branch, earlier callback-state initialization, retained commented Python sensor-path experiment. |
| [Privileged agent](../src/tum_agents/autoware_agent/aw_priviliged.py) | Ground-truth localization and object publication, routing and control infrastructure | Vehicle-model propagation, LiDAR sensor definitions, per-sensor Python conversion loop, optional camera placement. Rear LiDAR is not published by the new loop. |
| [C++ bridge](../src/autoware_carla_cpp_bridge) | Entire bridge implementation and launch configuration | Integrated as an existing dependency; no local implementation changes in the contribution commit. Its Git tree is identical to upstream base (`c45e39cf0d01ddbc1897aec8a98215abcdc70ab5`). |
| [Evaluation configuration](../config/config.yaml) | Framework configuration schema | Selected FR-max label and Town01 v2 route; these values do not themselves implement external ego spawning. |
| [Cyclist routes](SIMULATION_SCENARIOS.md) | External route parser, behavior trees, scenario classes | Three new route XML files with waypoints, trigger locations, bicycle intent, weather, and optional traffic parameters. |

Commit `87815a1` adds the prior short setup README. The portfolio documentation adds explicit architecture, reproducibility instructions, and evidence boundaries; it does not turn configuration intent into measured results.

## Supporting repositories

[carla_yuhesen_fr_max](https://github.com/gokcerSnmzck/carla_yuhesen_fr_max) was introduced in commit `552b29b` by Gökcer. It supplies five FR-max-named ROS 2 packages with geometry, sensor transforms, Xacro, mesh references, simulation parameters, and launch composition. Vehicle package manifests identify Gökcer as maintainer. Sensor manifests still identify Gemb Kaljavesi; launch files include TUM and Tier IV copyright notices. The defensible contribution is **platform adaptation and integration**; a single import commit cannot establish original authorship of every line. No earlier supporting-package baseline is recorded locally.

[carla_fr_maxv10](https://github.com/gokcerSnmzck/carla_fr_maxv10) was introduced in commit `34cfc05` by Gökcer. It packages 34 LFS-managed Unreal asset files. Blueprint, physics, mesh/skeleton, animation and material filenames identify their integration roles. Asset publication and model integration are supported; original CAD, mesh, material, and Unreal authoring provenance is not established by this commit. Retain third-party rights and obtain asset provenance before redistribution beyond the existing project terms.

## What the implementation enables

The artifacts provide a platform-specific starting point for studying a compact Ackermann robot within an Autoware/CARLA loop: converting vehicle commands, describing geometry and sensors, and defining repeatable following experiments. They demonstrate interface-level systems engineering across Python, ROS 2/C++, YAML, Xacro, XML, and simulator content.

Control stability, localization accuracy, perception quality, collision avoidance, cyclist-following completion, and deployment on physical FR-max hardware require separate measured evidence. No benchmark or successful end-to-end run is claimed here.

## Upstream research attribution

The original framework requests citation of its preprint. Retain this attribution when using AWCL in research:

```bibtex
@inproceedings{awcl_26,
  title = {Empirical mapping of technical bottlenecks in autonomous driving: A cross-platform evaluation of simulated and real-world performance},
  author = {Kaljavesi, Gemb and Majstorovic, Domagoj and Clement, Maxime and Diermeyer, Frank},
  year = {2026},
  howpublished = {Preprint}
}
```
