# Autoware CARLA Leaderboard (AWCL)
This repo is sourced from the original project --> https://github.com/TUMFTM/autoware_carla_leaderboard
Yuhesen FR-max model vehicle scenario execution is added into the agents.

1) Run the CARLA Simulator: ./CarlaUE4.sh --ros2
2) Run the C++ Bridge: ros2 launch autoware_carla_cpp_bridge aw_carla_cpp_bridge.launch.py
3) Start the Autoware: ros2 launch autoware_launch e2e_simulator.launch.xml vehicle_model:=yuhesen_fr_max_vehicle sensor_model:=yuhesen_fr_max_sensor_kit map_path:=/home/goekcer.soenmezocak001/carla_leaderboard/Town01
4) Initiate the scenario: python3 src/external/leaderboard/leaderboard/leaderboard_evaluator.py --conf_file_path=config/config.yaml

For step 1, find the FR-max model files here --> https://github.com/gokcerSnmzck/carla_fr_maxv10
For step 3, find the FR-max description files here --> https://github.com/gokcerSnmzck/carla_fr_maxv10
