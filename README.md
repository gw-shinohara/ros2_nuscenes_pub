# ros2_nuscenes_pub

**nuScenes Dataset Streaming & Dense Depth Estimation with ROS 2**

このリポジトリは、[nuScenesデータセット](https://www.nuscenes.org/)のデータをROS 2トピックとしてストリーミング配信し、さらに[Depth Anything V2](https://github.com/DepthAnything/Depth-Anything-V2)を用いて高精度かつLiDAR点群とスケール整合の取れた密な深度画像（Dense Depth）をリアルタイムに生成・配信するROS 2パッケージです。

## 主な機能

1.  **nuScenes Streamer**:
    * nuScenesデータセット（画像、LiDAR点群、Ego Pose、キャリブレーション情報）を読み込み、ROS 2標準メッセージで配信します。
    * LiDAR点群をカメラ画像平面に投影し、疎な深度画像（Sparse Depth）として配信します。
    * `/tf_static` を使用して、各センサー間の静的な座標変換を配信します。
    * [cite: `nuscenes_pkg/nuscenes_streamer.py`]

2.  **Depth Enhancer**:
    * 配信されたカメラ画像を入力として、**Depth Anything V2** モデルを用いて相対的な深度を推論します。
    * LiDARから得られた疎な深度情報（Sparse Depth）を利用して、推論された相対深度を絶対距離（メートル単位）にスケーリング補正します。これにより、画像全体で密（Dense）かつ実スケールの深度マップが得られます。
    * [cite: `nuscenes_pkg/depth_enhancer.py`]

## 必要要件 (Requirements)

* **OS**: Ubuntu 20.04 / 22.04 (推奨)
* **ROS 2**: Humble / Iron / Jazzy (Pythonクライアント `rclpy` 依存)
* **Python Dependencies**:
    * `nuscenes-devkit`
    * `torch` (CUDA対応推奨)
    * `opencv-python`
    * `pyquaternion`
    * `Pillow`

```bash
pip install nuscenes-devkit torch opencv-python pyquaternion Pillow
```

## インストール

1.  ワークスペースの作成とクローン
    ```bash
    mkdir -p ~/ros2_ws/src
    cd ~/ros2_ws/src
    git clone [https://github.com/gw-shinohara/ros2_nuscenes_pub.git](https://github.com/gw-shinohara/ros2_nuscenes_pub.git)
    ```

2.  ビルド
    ```bash
    cd ~/ros2_ws
    colcon build --symlink-install --packages-select nuscenes_pkg
    source install/setup.bash
    ```

## 準備

### 1. nuScenes データセット
[nuScenes公式サイト](https://www.nuscenes.org/download)からデータセット（例: `v1.0-mini`）をダウンロードし、展開してください。

例: `/root/data/nuscenes/v1.0-mini`

### 2. Depth Anything V2 重みファイル
[Depth Anything V2のリポジトリ](https://github.com/DepthAnything/Depth-Anything-V2) またはHugging Faceからモデルの重みファイル（例: `depth_anything_v2_vits.pth`）をダウンロードしてください。

例: `/root/weights/nuscenes_pkgs/Depth-Anything-V2-Small/depth_anything_v2_vits.pth`

## 実行方法

### 1. ストリーミングのみ実行 (Streaming Only)
nuScenesのデータを再生し、画像と疎な深度（LiDAR投影）のみを配信します。

```bash
ros2 launch nuscenes_pkg streaming.launch.py \
    dataroot:=/path/to/nuscenes/v1.0-mini \
    version:=v1.0-mini \
    scene_names:="['scene-0061']"
```

### 2. Dense Depth 生成を含めて実行 (With Depth Enhancer)
ストリーミングに加え、GPUを使用して密な深度画像を生成・配信します。**`checkpoint_path` の指定が必須です。**

```bash
ros2 launch nuscenes_pkg streaming_w_dense_depth.launch.py \
    dataroot:=/path/to/nuscenes/v1.0-mini \
    checkpoint_path:=/path/to/depth_anything_v2_vits.pth \
    publish_rate_hz:=0.5
```
*注意: Depth Anything V2の推論は計算コストが高いため、`publish_rate_hz` を低め（0.5Hz〜数Hz程度）に設定することを推奨します。*

## パラメータ詳細

| パラメータ名 | デフォルト値 | 説明 |
| :--- | :--- | :--- |
| `dataroot` | `/root/data/...` | nuScenesデータセットのルートパス |
| `version` | `v1.0-mini` | 使用するデータセットのバージョン |
| `scene_names` | `['scene-0061']` | 再生するシーン名のリスト |
| `play_all_scenes`| `False` / `True` | `True`の場合、`scene_names`を無視して全シーンを再生 |
| `publish_rate_hz`| `10.0` or `0.5` | 再生速度（Hz）。Dense Depth生成時は処理落ちを防ぐため下げてください |
| `lidar_sensor` | `LIDAR_TOP` | 使用するLiDARセンサーのチャンネル名 |
| `checkpoint_path`| (必須) | Depth Anything V2の `.pth` 重みファイルのパス |

[cite: `launch/streaming.launch.py`, `launch/streaming_w_dense_depth.launch.py`]

## トピック一覧

各カメラ (`CAM_FRONT`, `CAM_FRONT_LEFT`, `CAM_BACK` 等) に対して以下のトピックが発行されます。

* **`/nuscenes/<camera>/image_raw`** (`sensor_msgs/Image`): RGB画像
* **`/nuscenes/<camera>/camera_info`** (`sensor_msgs/CameraInfo`): カメラ内部パラメータ
* **`/nuscenes/<camera>/depth`** (`sensor_msgs/Image`): LiDAR点群を投影した疎な深度画像 (32FC1)
* **`/nuscenes/<camera>/dense_depth`** (`sensor_msgs/Image`): Depth Anything V2で生成・補正された密な深度画像 (32FC1)

その他:
* **`/nuscenes/odometry`** (`nav_msgs/Odometry`): 車両のオドメトリ
* **`/tf_static`**: 車両座標系 (`base_link`) と各センサー間のTF

[cite: `nuscenes_pkg/nuscenes_streamer.py`, `nuscenes_pkg/depth_enhancer.py`]

## ライセンス

このパッケージは **Apache License 2.0** の下で公開されています。
詳細は `LICENSE` ファイルを参照してください。

[cite: `package.xml`, `LICENSE`]

## 謝辞

* This project uses [nuScenes devkit](https://github.com/nutonomy/nuscenes-devkit).
* Dense depth estimation relies on [Depth Anything V2](https://github.com/DepthAnything/Depth-Anything-V2).
* Built with ROS 2.
