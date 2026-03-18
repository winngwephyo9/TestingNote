# --- Step 5: Classificationのマッピング修正 ---
def map_kitti_to_asprs(labels):
    asprs_labels = np.zeros_like(labels)
    mapping = {
        0: 0,    # never classified
        10: 8,   # car -> Key-point (8番をCarとして利用)
        40: 11,  # road -> Road (11番)
        44: 11,  # sidewalk -> Road (11番)
        50: 6,   # building -> Building (6番)
        51: 6,   # wall -> Building (6番)
        70: 5,   # vegetation -> High Veg (5番)
        71: 4,   # trunk -> Medium Veg (4番)
        72: 2,   # terrain -> Ground (2番)
    }
    for kitti_id, asprs_id in mapping.items():
        asprs_labels[labels == kitti_id] = asprs_id
    return asprs_labels

# ここを必ず有効にする
las.classification = map_kitti_to_asprs(down_labels).astype(np.uint8)

# --- Step 6: 色の割り当て修正 (RGBA表示用) ---
color_map = {
    6: [255, 165, 0],   # Building: Orange
    11: [128, 128, 128], # Road: Gray
    8: [255, 0, 0],      # Car: Red
    5: [0, 128, 0],      # Veg: Green
    2: [139, 69, 19],    # Ground: Brown
}
# (以下、色の代入処理は前回のコードと同様)


Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
    let pointcloud = e.pointcloud;
    let material = pointcloud.material;
    viewer.scene.addPointCloud(pointcloud);
    
    material.activeAttributeName = "classification";

    // --- 分類名と色のカスタマイズ ---
    // 番号とラベルの紐付けを上書き
    viewer.setClassColor(6, new THREE.Color(1, 0.64, 0)); // Building -> Orange
    viewer.setClassColor(11, new THREE.Color(0.5, 0.5, 0.5)); // Road -> Gray
    viewer.setClassColor(8, new THREE.Color(1, 0, 0)); // Car -> Red
    
    // Potreeの内部分類名を変更（UIに反映させるため）
    Potree.Classification = {
        0: { visible: true, name: 'Never Classified', color: [0.5, 0.5, 0.5, 1] },
        2: { visible: true, name: 'Ground', color: [0.6, 0.3, 0.1, 1] },
        5: { visible: true, name: 'Vegetation', color: [0, 0.5, 0, 1] },
        6: { visible: true, name: 'Building', color: [1, 0.64, 0, 1] },
        8: { visible: true, name: 'Car', color: [1, 0, 0, 1] },
        11: { visible: true, name: 'Road', color: [0.3, 0.3, 0.3, 1] }
    };

    // UIを再描画（サイドバーに反映）
    viewer.gui.sidebar.initClassification();

    material.size = 1;
    material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
    viewer.fitToScreen();
});


new

import os
import torch
import numpy as np
import open3d.ml.torch as ml3d
import open3d.ml as ml
import laspy

def main():
    base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"
    cfg_path = os.path.join(base_path, "ml3d/configs/randlanet_semantickitti.yml")
    ckpt_path = os.path.join(base_path, "randlanet_semantickitti_202201071330utc.pth")
    input_las = os.path.join(base_path, "3d_point_data/tokyo_data.las")
    output_las = os.path.join(base_path, "3d_point_data/tokyo_segmented_detailed.las")

    # --- Step 1: モデル準備 ---
    cfg = ml.utils.Config.load_from_file(cfg_path)
    model = ml3d.models.RandLANet(**cfg.model)
    pipeline = ml3d.pipelines.SemanticSegmentation(model)
    pipeline.load_ckpt(ckpt_path)

    # --- Step 2: データ読み込み ---
    las = laspy.read(input_las)
    points = np.asarray(las.xyz, dtype=np.float32)
    points_normalized = points - np.mean(points, axis=0)

    input_dict = {
        'point': points_normalized,
        'feat': np.zeros((len(points), 0), dtype=np.float32), 
        'label': np.zeros((len(points),), dtype=np.int32)
    }

    # --- Step 3: AI推論 ---
    pipeline.model.eval()
    with torch.no_grad():
        result = pipeline.run_inference(input_dict)
    kitti_labels = result['predict_labels']

    # --- Step 4: 詳細マッピング (SemanticKITTI -> LAS/Potree Standard) ---
    # Potreeのサイドバーにある項目に連動するように番号を振り直します
    las_labels = np.zeros_like(kitti_labels, dtype=np.uint8)

    # マッピング辞書 {SemanticKITTI ID : Potree/LAS ID}
    mapping = {
        0: 0,    # unlabeled -> never classified
        1: 1,    # outlier -> unclassified
        10: 9,   # car -> water (Potreeで他と被らない予備番号として利用可)
        11: 9,   # bicycle
        13: 11,  # road -> road
        15: 11,  # sidewalk -> road
        16: 5,   # vegetation -> high vegetation
        18: 6,   # building -> building
        19: 5,   # other-ground -> high vegetation
        20: 12,  # fence -> overlap (予備)
        30: 2,   # pole -> ground (あえてgroundに近い色にするか、1番へ)
        40: 1,   # road-marking
    }

    for kitti_id, las_id in mapping.items():
        las_labels[kitti_labels == kitti_id] = las_id

    # --- Step 5: 保存処理 ---
    new_las = laspy.create(point_format=las.header.point_format, file_version=las.header.version)
    new_las.header.offsets = las.header.offsets
    new_las.header.scales = las.header.scales
    new_las.x, new_las.y, new_las.z = las.x, las.y, las.z

    # Classification書き込み（これがフィルタリングの鍵です）
    new_las.classification = las_labels

    # Potreeでの視認性を高めるためのRGB色設定 (16bit)
    colors = np.zeros((len(las_labels), 3), dtype=np.uint16)
    # クラスごとの色定義 [R, G, B]
    color_map = {
        6: [255, 255, 0],   # Building: Yellow
        5: [0, 255, 0],     # Vegetation: Green
        11: [60, 60, 60],   # Road: Dark Gray
        9: [255, 0, 0],     # Vehicle: Red
        2: [150, 75, 0],    # Ground: Brown
    }
    
    for cid, rgb in color_map.items():
        mask = (las_labels == cid)
        colors[mask] = np.array(rgb, dtype=np.uint16)

    new_las.red = colors[:, 0] << 8
    new_las.green = colors[:, 1] << 8
    new_las.blue = colors[:, 2] << 8

    new_las.write(output_las)
    print(f"詳細分類完了: {output_las}")

if __name__ == "__main__":
    main()

import numpy as np
import laspy
import os

# --- 設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/0b_segmented_result_fixed.laz"

os.makedirs(os.path.dirname(output_laz), exist_ok=True)

# --- データ読み込み ---
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3].astype(np.float64)
# SemanticKITTIラベル取得
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# --- ダウンサンプリング ---
voxel_size = 0.01
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)
down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

# --- LASヘッダー ---
header = laspy.LasHeader(point_format=7, version="1.4")
header.offsets = np.min(down_xyz, axis=0)
header.scales = [0.001, 0.001, 0.001]
las = laspy.LasData(header)
las.x, las.y, las.z = down_xyz[:, 0], down_xyz[:, 1], down_xyz[:, 2]

# --- Step 5: Classificationのマッピング (ココが重要) ---
# Potreeのチェックボックス(ASPRS規格)に合わせる
def map_kitti_to_asprs(labels):
    asprs_labels = np.zeros_like(labels)
    mapping = {
        0: 0,    # never classified
        10: 1,   # car -> Unclassified(1)
        40: 11,  # road -> Road(11)
        44: 11,  # sidewalk -> Road(11)
        50: 6,   # building -> Building(6) ★重要
        70: 5,   # vegetation -> High Veg(5)
        71: 4,   # trunk -> Medium Veg(4)
        72: 3,   # terrain -> Low Veg(3)
    }
    for kitti_id, asprs_id in mapping.items():
        asprs_labels[labels == kitti_id] = asprs_id
    return asprs_labels

# 修正：マッピングした後の値を代入
las.classification = map_kitti_to_asprs(down_labels).astype(np.uint8)

# --- Step 6: 色の割り当て (赤色=建物に設定) ---
# クラスIDごとの色設定 [R, G, B]
color_map = {
    6: [255, 0, 0],    # Building: Red (ご要望通り)
    11: [128, 128, 128], # Road: Gray
    5: [0, 255, 0],    # Veg: Green
    1: [255, 255, 0],  # Car/Other: Yellow
    0: [100, 100, 100] # Default
}

# デフォルトはグレー
colors = np.full((len(las.classification), 3), [100, 100, 100], dtype=np.uint16)
for cid, rgb in color_map.items():
    colors[las.classification == cid] = rgb

# laspyは16bitカラーなので256倍する
las.red = colors[:, 0] * 256
las.green = colors[:, 1] * 256
las.blue = colors[:, 2] * 256

las.update_header()
las.write(output_laz)
print(f"変換成功: {output_laz}")


Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {
    let pointcloud = e.pointcloud;
    let material = pointcloud.material;
    viewer.scene.addPointCloud(pointcloud);
    
    // --- 修正ポイント ---
    // 1. 最初から属性を「Classification」に設定
    material.activeAttributeName = "classification"; 
    
    // 2. クラスごとの色をJS側でも定義（赤色=建物にする場合）
    // ASPRS Class 6 (Building) を 赤色に設定
    viewer.setClassColor(6, new THREE.Color(1, 0, 0)); 
    // ASPRS Class 11 (Road) を グレーに設定
    viewer.setClassColor(11, new THREE.Color(0.5, 0.5, 0.5));
    
    material.size = 1;
    material.pointSizeType = Potree.PointSizeType.ADAPTIVE;
    material.shape = Potree.PointShape.SQUARE;
    viewer.fitToScreen();
});

点群専用データのコードを実行して試してみると、classificationの時建物の部分はgreenとyellow二つの色がついています。そのため、greenはmedium vegetationの色になっています。建物なのにこの色が付いているので変なclassificationになっています。後左側のclassificationにroad,car,なども追加して
<img width="1387" height="961" alt="image" src="https://github.com/user-attachments/assets/7dcd1f22-e637-4e82-b0b2-a103044a36b7" />
<img width="1285" height="941" alt="image" src="https://github.com/user-attachments/assets/1a621736-7ba7-4e7b-a89a-88d87ff18776" />

tokyo dataset はrgbaの時に建物の色がgray,yellow,redなどがついています。なぜですか。
classification時に建物の色がdefault color, yellow is building color and blue is water colorになっています。これも変な形になっています。修正してください。
<img width="1659" height="931" alt="image" src="https://github.com/user-attachments/assets/c6f2fba3-8820-45a0-9c65-09e6d86e6469" />
<img width="1621" height="967" alt="image" src="https://github.com/user-attachments/assets/0806f9d1-2950-43e1-9b16-6aca7f69bb87" />





