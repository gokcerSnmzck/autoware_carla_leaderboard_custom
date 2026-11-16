# Engineering challenges and implementation

## Adapting passenger-car control to a compact robot

**Challenge:** the inherited steering conversion uses an Audi-specific nonlinear table reaching 58.6798°. Applying the same mapping to a smaller steering envelope would misrepresent the requested wheel angle.

**Approach and implementation:** `AutowareToCarlaControl(vehicle_model=...)` selects an FR-max table from 0° to 25° in 2.5° increments. `convert_control()` converts the Autoware tire angle from radians to degrees; interpolation maps magnitude to 0…−1 and applies the input sign, with saturation at ±25°. This retains the inherited coordinate-sign convention while changing the platform envelope.

**Result:** a dedicated FR-max conversion path in [aw_converter.py](../src/tum_agents/autoware_agent/aw_converter.py), activated by an actual actor type containing `fr-max`. Agreement with wheel physics and the physical steering actuator remains a simulation/hardware measurement task.

## Configuring longitudinal response

**Challenge:** a compact robot needs its own acceleration-loop parameters rather than inheriting passenger-car actuator limits.

**Approach and implementation:** the FR-max branch sets `Kp=0.25`, `Ki=0.05`, `Kd=0`, throttle ≤0.4, brake ≤0.3, and a fixed 0.05 s controller interval. The inherited controller integrates acceleration error and undoes the latest integration increment when saturated. The baseline Audi values are 0.5/0.1/0, throttle ≤0.6, brake ≤0.2.

**Result:** explicit robot-specific gains and command bounds. Comments mention a 156 kg robot and 1800 W motor as tuning rationale; these values have not been independently validated against the asset physics or hardware specifications. Lower gains are an engineering choice, not evidence that oscillation or stopping distance improved. The fixed controller interval is not a measured simulation rate.

## Aligning sensors and platform geometry

**Challenge:** the robot's footprint and low sensor mounts differ from the baseline vehicle.

**Approach and implementation:** the E2E FR-max branch requests front/rear LiDAR at x=±0.75 m, z=0.55 m with 32/16 channels, plus GNSS, IMU, and an optional front camera. The supporting package encodes geometry, a rear-axle `base_link`, kit-relative extrinsics and crop-filter bounds.

**Result:** configurable robot-specific sensor and geometry artifacts across the agent and Autoware packages. The +0.425 m kit offset means equal-looking local coordinates are not automatically equal world transforms. The bridge still uses 128/64 channels, and Xacro uses VLS-128 macros. These differences require an explicit calibration/channel decision before execution; no calibration values were silently changed.

## Initializing callback state

**Challenge:** ROS subscriptions and a background spin thread can observe partially initialized E2E state.

**Approach and implementation:** the custom commit initializes time, vehicle identity, control queue, route flags, and camera state before creating ROS resources and starting the thread.

**Result:** the relevant E2E state now exists before callback execution can begin. Runtime startup and stress testing remain necessary; the change does not establish freedom from all races.

## Reproducible research inputs

**Challenge:** cyclist-following experiments need controlled route and traffic inputs across maps.

**Approach and implementation:** three XML routes provide trigger positions, waypoints, bicycle intent, and a traffic-parametrized Town01 straight variant.

**Result:** reviewable scenario definitions. The declared external scenario class ignores the intended bicycle parameters, so completing that integration requires a separately approved dependency change. [Scenario semantics and capture plan](SIMULATION_SCENARIOS.md) define how to demonstrate the result without inventing test outcomes.
