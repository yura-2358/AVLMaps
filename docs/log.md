# 実機作業 生ログ

**※2026-10-22注記**: 本ファイルは2026-10-22にsobit_edu実機の`ros2 topic list`出力で上書き済み。旧内容（HSR実機ログ: 2026-09-07接続・`hsr_data_collector.py`収集・nav2起動等）は消失、docs/はgit未追跡のため復元不可。他docsの「詳細docs/log.md」参照（strategy.md・step1.md・step2.md・resolved.md等）はHSRログ指す→参照先不在。要点は各参照元・resolved.mdに抽出済み。

以降、sobit_edu実機ログを追記。

## 2026-10-22 sobit_edu `real_minimal.launch.py`起動後 topic list

```
hark_ws yura@:~$ ros2 topic list
/clicked_point
/diagnostics
/goal_pose
/initialpose
/parameter_events
/rosout
/sobit_edu/commands/controller_info
/sobit_edu/commands/digital_output
/sobit_edu/commands/external_power
/sobit_edu/commands/led1
/sobit_edu/commands/led2
/sobit_edu/commands/motor_power
/sobit_edu/commands/reset_odometry
/sobit_edu/commands/sound
/sobit_edu/commands/velocity
/sobit_edu/controller_info
/sobit_edu/controller_manager/activity
/sobit_edu/controller_manager/introspection_data/full
/sobit_edu/controller_manager/introspection_data/names
/sobit_edu/controller_manager/introspection_data/values
/sobit_edu/controller_manager/statistics/full
/sobit_edu/controller_manager/statistics/names
/sobit_edu/controller_manager/statistics/values
/sobit_edu/debug/raw_control_command
/sobit_edu/debug/raw_data_command
/sobit_edu/debug/raw_data_stream
/sobit_edu/depth_registered/points
/sobit_edu/events/bumper
/sobit_edu/events/button
/sobit_edu/events/cliff
/sobit_edu/events/digital_input
/sobit_edu/events/power_system
/sobit_edu/events/robot_state
/sobit_edu/events/wheel_drop
/sobit_edu/head_camera/depth/camera_info
/sobit_edu/head_camera/depth/image/compressed
/sobit_edu/head_camera/depth/image/compressedDepth
/sobit_edu/head_camera/depth/image/theora
/sobit_edu/head_camera/depth/image/zstd
/sobit_edu/head_camera/depth_raw/camera_info
/sobit_edu/head_camera/depth_raw/image
/sobit_edu/head_camera/depth_raw/image/compressed
/sobit_edu/head_camera/depth_raw/image/compressedDepth
/sobit_edu/head_camera/depth_raw/image/theora
/sobit_edu/head_camera/depth_raw/image/zstd
/sobit_edu/head_camera/depth_registered/image_raw
/sobit_edu/head_camera/depth_registered/points
/sobit_edu/head_camera/ir/camera_info
/sobit_edu/head_camera/ir/image_raw
/sobit_edu/head_camera/ir/image_raw/compressed
/sobit_edu/head_camera/ir/image_raw/compressedDepth
/sobit_edu/head_camera/ir/image_raw/theora
/sobit_edu/head_camera/ir/image_raw/zstd
/sobit_edu/head_camera/projector/camera_info
/sobit_edu/head_camera/rgb/camera_info
/sobit_edu/head_camera/rgb/image_raw
/sobit_edu/head_camera/rgb/image_raw/compressed
/sobit_edu/head_camera/rgb/image_raw/compressedDepth
/sobit_edu/head_camera/rgb/image_raw/theora
/sobit_edu/head_camera/rgb/image_raw/zstd
/sobit_edu/joint_states
/sobit_edu/laser_status
/sobit_edu/odom
/sobit_edu/robot_description
/sobit_edu/scan
/sobit_edu/sensors/battery_state
/sobit_edu/sensors/core
/sobit_edu/sensors/dock_ir
/sobit_edu/sensors/imu_data
/sobit_edu/sensors/imu_data_raw
/sobit_edu/version_info
/tf
/tf_static
```
