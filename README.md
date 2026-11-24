# waypoint_manager

ROS 2 (Humble) 向けのシンプルなウェイポイントフォロワーです。`waypoint_editor` と共通の CSV 形式で定義したウェイポイント列に沿って、次の目標姿勢を `goal_pose` へ順番に送信します。

速度指令の送受信は行いません。ナビゲーションは [value_iteration2](https://github.com/kei487/value_iteration2/tree/mugimaru2026)（`goal_pose` 購読・`/vi_status` 配信）、速度合成は [velocity_manager](https://github.com/CIT-Autonomous-Robot-Lab/velocity_manager) を想定しています。

## パッケージ構成

| パッケージ | 説明 |
|-----------|------|
| `waypoint_manager` | メインノード (`waypoint_manager`) |
| `waypoint_manager_msgs` | ウェイポイント定義用カスタムメッセージ |

## 動作概要

1. 起動時に CSV ファイルからウェイポイントを読み込む
2. `/vi_status` が `idle` になるまで待機し、先頭ウェイポイントを `goal_pose` に publish
3. 500 ms 周期のタイマーでロボット位置（`map` フレーム）を監視
4. 現在のウェイポイント半径内に入ると、次のウェイポイントへ目標を更新
5. 送信後も `/vi_status` が `idle` のままなら、`goal_resend_interval_sec` 間隔で同一目標を再送
6. `robot_wait: true` のウェイポイント到達後は goal 再送を抑制（速度停止は velocity_manager 側）
7. `/restart_waypoint_manager` サービスで先頭ウェイポイントから再開（再び `/vi_status` idle 待ち）

```
  CSV waypoints (waypoint_editor 共通形式)
        |
        v
  waypoint_manager ----goal_pose----> value_iteration2 ----cmd_vel----> velocity_manager
        ^                     |
        |                     v
   TF (map<-base)         /vi_status (idle / ...)
        |
        +---- /vi_status <----+
```

## 依存関係

- ROS 2 Humble
- `rclcpp`, `geometry_msgs`, `std_msgs`, `std_srvs`
- `nav2_util`, `tf2_ros`
- `waypoint_manager_msgs`

## ビルド

```bash
cd ~/cursor_ws/waypint_ws   # ワークスペースルート
source /opt/ros/humble/setup.bash
colcon build --packages-select waypoint_manager_msgs waypoint_manager
source install/setup.bash
```

## 起動

```bash
ros2 launch waypoint_manager waypoint_manager.launch.xml
```

別のウェイポイントファイルを使う場合:

```bash
ros2 launch waypoint_manager waypoint_manager.launch.xml \
  waypoint_csv:=/path/to/waypoint.csv
```

パラメータのみ指定して起動する場合:

```bash
ros2 run waypoint_manager waypoint_manager --ros-args \
  -p waypoint_csv_path:=$(ros2 pkg prefix waypoint_manager)/share/waypoint_manager/config/waypoint_test.csv \
  -p waypoint_radius:=0.5 \
  -p goal_resend_interval_sec:=3.0
```

## パラメータ

| 名前 | 型 | デフォルト | 説明 |
|------|-----|-----------|------|
| `waypoint_csv_path` | string | `waypoint.csv` | ウェイポイント定義 CSV のパス |
| `waypoint_radius` | double | `0.5` | 各ウェイポイントの到達判定半径（m）。CSV で個別指定がない場合に使用 |
| `goal_resend_interval_sec` | double | `3.0` | `/vi_status` が `idle` のままのとき同一 `goal_pose` を再送する間隔（秒） |

## トピック

| トピック | 型 | 方向 | 説明 |
|---------|-----|------|------|
| `goal_pose` | `geometry_msgs/PoseStamped` | publish | 現在のナビゲーション目標（value_iteration2 が購読） |
| `/vi_status` | `std_msgs/String` | subscribe | value_iteration2 の状態。`idle` のとき初回送信・再送の条件に使用 |

## サービス

| サービス | 型 | 説明 |
|---------|-----|------|
| `/restart_waypoint_manager` | `std_srvs/Trigger` | ウェイポイント列を先頭から再開し、待機状態を解除（送信は `/vi_status` idle 待ち） |

```bash
ros2 service call /restart_waypoint_manager std_srvs/srv/Trigger
```

## ウェイポイント CSV 形式（`waypoint_editor` 共通）

```csv
id,pose_x,pose_y,pose_z,rot_x,rot_y,rot_z,rot_w,waypoint_radius,robot_wait,command,
0,6.13,3.99,0,0,0,0.707,0.707,0.5,false,,
1,4.68,6.62,0,0,0,0.383,0.924,0.5,true,,
2,2.19,7.05,0,0,0,0.0,1.0,0.5,false,wait,
```

### フィールド説明

- `id`: ウェイポイント識別子（ログ表示用）
- `pose_x/y/z`, `rot_x/y/z/w`: `map` フレーム上の位置とクォータニオン
- `waypoint_radius`: 到達判定半径 [m]（省略時はパラメータ `waypoint_radius`）
- `robot_wait`: 到達後に goal 再送を抑制するか（`true`/`false`）。速度停止は velocity_manager 側。再開は `/restart_waypoint_manager`
- `command`: `waypoint_editor` 用の任意コマンド列（本ノードでは未使用）

`waypoint_radius` / `robot_wait` 列が無い従来形式の CSV も読み込めます（半径はパラメータ値、`robot_wait` は `false`）。

サンプルファイル:

- `waypoint_manager/config/waypoint_test.csv` — 3 点のテスト経路
- `waypoint_manager/config/tsukuba/waypoint_tc2026_1.csv` — つくばチャレンジ用経路
- `waypoint_manager/config/tsudanuma/waypoint*.csv` — 津田沼用経路

### 旧 YAML からの変換

以前の YAML コンフィグは次のスクリプトで CSV に変換できます（`python3-yaml` / PyYAML が必要）:

```bash
# ワークスペース内のスクリプトを直接実行
./src/waypoint_manager/waypoint_manager/scripts/yaml_to_csv.sh /path/to/waypoint.yaml

# 出力先を指定
./src/waypoint_manager/waypoint_manager/scripts/yaml_to_csv.sh \
  -o /path/to/waypoint.csv /path/to/waypoint.yaml

# 複数ファイル（各入力と同名の .csv を生成）
./src/waypoint_manager/waypoint_manager/scripts/yaml_to_csv.sh -r 0.5 a.yaml b.yaml

# インストール後
ros2 run waypoint_manager yaml_to_csv.sh /path/to/waypoint.yaml
```

変換内容:

- `position` + `euler_angle.z` → `pose_*` / `rot_*`（yaw からクォータニオンへ）
- `functions.variable_waypoint_radius` → `waypoint_radius`（未指定時は `-r` の値）
- `robot_wait` → `robot_wait`（未指定時は `false`）

## 必要条件（実行環境）

- `map` → `base_link`（または `base_footprint`）の TF が配信されていること
- value_iteration2 が `goal_pose` を受け取り、`/vi_status` を配信すること
- 速度指令は velocity_manager 側で扱うこと

TF が無い場合、位置取得に失敗しウェイポイント到達判定は行われません（ログスパムは抑止済み）。
`/vi_status` が `idle` にならない場合、初回の `goal_pose` は送信されません。

## ライセンス

Apache-2.0

## メンテナ

Keitaro Nakamura (numerugon487@gmail.com)
