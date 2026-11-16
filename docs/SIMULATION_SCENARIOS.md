# Cyclist-following scenario design

These routes define research inputs for evaluating following behavior with a custom Ackermann robot. Autoware supplies ego planning and control; the XML does not implement a following controller.

## Routes and triggers

| Definition | Waypoints | Trigger and configured lead actor |
|---|---|---|
| [town01_cyclist_follower.xml](../resources/routes/town01_cyclist_follower.xml) | 10 positions; starts near (88.410, 261.330), turns east, ends near (299.620, 330.610) | `FollowLeadingVehicle` at (124.620, 330.592), yaw ≈0°; bicycle `vehicle.diamondback.century`, distance 15, velocity 5. |
| [town01_cyclist_follower_v2.xml](../resources/routes/town01_cyclist_follower_v2.xml) | 8 positions; eastbound straight from (100.620, 330.592) to (299.620, 330.610) | Lead trigger at (110.620, 330.592), distance 10, velocity 5; traffic trigger at the start. Default selected route. |
| [town10_follow_cyclist.xml](../resources/routes/town10_follow_cyclist.xml) | 3 positions in **Town10HD**, from (86.580, −13.240) through (76.580, −13.240) to (−50, −13.240) | Trigger at the middle waypoint, yaw 180°; same bicycle, distance 10, velocity 5. |

All three set low-cloud, dry weather and a 45° sun altitude. Route positions are CARLA world coordinates; they are interpolated by the external route tooling. Triggers must fall close enough to the interpolated route or the external route scenario skips them.

## Background traffic

Town01 v2 adds `BackgroundActivityParametrizer`. It configures front/back road vehicle counts to zero, opposite maximum actors to zero and `opposite_active=0`, and junction maximum actors to zero. Spawn/source distances and junction source percentage are also explicit. This expresses an isolation experiment; it does not prove that all background actors are absent in a running simulation.

The inspected scenario parametrizer reads these names through `get_parameter()`, which converts values to floats. The eventual effect depends on its background behavior consumers and route trigger execution.

## Actual dependency behavior

At Scenario Runner commit [`0ec3b378`](https://github.com/TUMFTM/scenario_runner/blob/0ec3b37897ac8c93af74e8dfeec5427a131f3acc/srunner/scenarios/follow_leading_vehicle.py), `FollowLeadingVehicle` fixes its lead actor to `vehicle.nissan.patrol`, initial displacement to 25 m, and speed to 10 m/s. It does not read the XML `model`, `distance`, or `velocity` elements. Therefore the bicycle and numeric values above describe **authored intent**, not verified actor behavior. Their units need to be defined by the adapted consumer; interpreting them as metres and m/s is the intended convention.

The inherited behavior follows waypoints, stops in relation to an intersection, and waits for the ego to approach. A “cyclist-following” filename does not change this implementation. An adapted scenario class and a Leaderboard ego-spawning patch must be reviewed before these definitions can demonstrate FR-max following a cyclist. Neither dependency is modified by this documentation update.

## Selection and execution

In `config/config.yaml`, set `evaluation.simulation_setup.routes` to a route above; retain the town's matching Autoware map. `routes-subset` selects route IDs and `repetitions` controls repetitions. Launch with the evaluator command in [Setup](SETUP.md#launch-sequence). The checked-in default is Town01 v2, route ID `0`, one repetition, empty subset, Traffic Manager seed `0`.

## Demonstration capture checklist

- Record all three repository commits, CARLA server/client versions, Autoware commit, dependency patches/commits, route, map, seed, and controller/sensor settings.
- Capture CARLA showing the FR-max actor and the actual bicycle lead actor; save both runtime `type_id` values and blueprint-query output.
- Capture RViz with the FR-max mesh, TF axes, front/rear point clouds, localization, and planned trajectory. Verify frame and channel alignment first.
- Record a continuous following segment with simultaneous control/status topics and simulation clock; retain the rosbag or CARLA recording and result JSON.
- Report completion, collisions, following distance, and control response only when supported by the captured logs, with units, duration and evaluation method.
- Confirm publication rights, remove private paths or credentials, and add concise media captions identifying the exact experiment.

No run logs, scores, recordings, or completion reports were found in the three project repositories during the portfolio review. Nearby hardware-project media is separate evidence and must not be relabeled as CARLA results.
