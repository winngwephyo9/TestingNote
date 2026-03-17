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

    output_las = os.path.join(base_path, "3d_point_data/tokyo_segmented_5.las")



    # --- Step 1: モデル準備 ---

    cfg = ml.utils.Config.load_from_file(cfg_path)

    model = ml3d.models.RandLANet(**cfg.model)

    pipeline = ml3d.pipelines.SemanticSegmentation(model)

    pipeline.load_ckpt(ckpt_path)

    print(f"--- Step 1: モデル準備 ---")

    # --- Step 2: データ読み込み ---

    las = laspy.read(input_las)

    

    # XYZを一度に取り出す (Scale/Offset適用済み)

    points = np.asarray(las.xyz, dtype=np.float32)

    

    # 推論用に中心化

    mean_offset = np.mean(points, axis=0)

    points_normalized = points - mean_offset



    input_dict = {

        'point': points_normalized,

        'feat': np.zeros((len(points), 0), dtype=np.float32), 

        'label': np.zeros((len(points),), dtype=np.int32)

    }

    print(f"--- Step 2: データ読み込み ---")

    # --- Step 3: AI推論 ---

    pipeline.model.eval()

    with torch.no_grad():

        result = pipeline.run_inference(input_dict)

    labels = result['predict_labels']

    print(f"--- Step 3: AI推論 ---")

    # --- Step 4: 結果の保存 (RecursionError対策) ---

    # ヘッダーをコピーして新しいLASを作成

    # las.header.version を維持することでPotreeとの互換性を保つ

    new_las = laspy.create(point_format=las.header.point_format, file_version=las.header.version)

    

    # ヘッダーのオフセットとスケールを元データと完全に一致させる

    new_las.header.offsets = las.header.offsets

    new_las.header.scales = las.header.scales

    

    # 座標を直接コピー (整数値として正確に引き継ぐ)

    # これによりPotreeで「形が歪む」「浮く」問題が解消されます

    new_las.x = las.x

    new_las.y = las.y

    new_las.z = las.z



    # ラベルを適用

    new_las.classification = labels.astype(np.uint8)



    # SemanticKITTI カラーマップ (16bit対応)

    sem_kitti_cmap = np.array([

        [0, 0, 0], [100, 150, 245], [100, 230, 245], [30, 60, 150], [80, 30, 180],

        [100, 80, 250], [255, 30, 30], [255, 40, 200], [150, 30, 90], [255, 0, 255],

        [255, 150, 255], [75, 0, 75], [75, 0, 175], [0, 200, 255], [50, 120, 255],

        [0, 175, 0], [0, 60, 135], [80, 240, 150], [150, 240, 255], [0, 0, 255]

    ], dtype=np.uint16)



    valid_labels = np.clip(labels, 0, len(sem_kitti_cmap) - 1)

    rgb_colors = sem_kitti_cmap[valid_labels]



    # Potree用にカラーを16bitで書き込む

    new_las.red = rgb_colors[:, 0] << 8

    new_las.green = rgb_colors[:, 1] << 8

    new_las.blue = rgb_colors[:, 2] << 8



    # 保存

    new_las.write(output_las)

    print(f"修正完了: {output_las}")



if __name__ == "__main__":

    main()



.html file

    <script>

    

        window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));

        

        viewer.setEDLEnabled(true);

        viewer.setFOV(60);

        viewer.setPointBudget(2_000_000);

        document.title = "";

        viewer.setEDLEnabled(false);

        viewer.setBackground("gradient"); // ["skybox", "gradient", "black", "white"];

        viewer.setDescription(``);

        viewer.loadSettingsFromURL();

        

        viewer.setDescription("");

        

        viewer.loadGUI(() => {

            viewer.setLanguage('en');

            $("#menu_appearance").next().show();

            $("#menu_tools").next().show();

            $("#menu_clipping").next().show();

            viewer.toggleSidebar();

        });

        

        Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {

            let pointcloud = e.pointcloud;

            let material = pointcloud.material;

            viewer.scene.addPointCloud(pointcloud);

            //material.activeAttributeName = "rgba"; //[rgba, intensity, classification, ...]

            material.size = 1;

            material.pointSizeType = Potree.PointSizeType.ADAPTIVE;

            material.shape = Potree.PointShape.SQUARE;

            viewer.fitToScreen();

        });

        

    </script>



tokyo metroの点群データを.lasファイルで出力して、そのコードでclassificationして、web上に表示しました。その時に画像に表示通りにattributeがrgbaの時に建物の部分は緑色がついていますが、attributeがclassificationの時に建物の部分はdefaultの色が付いています。そのため、どうしたら正しくclassificationできるのか、filterのclassificationにbuilding,roadなどをselect/unselectしたら表示・非表示したいです。学習済のrandlanet_semantickittiを使ってclassificationしましたが、画像を見て、正しくclassificationしていますか。

完璧のコードで修正してください。
<img width="1709" height="976" alt="image" src="https://github.com/user-attachments/assets/06944661-cfea-4d17-9473-719132a4f2fd" />


次の点群データは専用データです。.e57formatなので

1.まずは .e57から.binの形で交換しました。

2.もらった.binをrandlanet_semantickittiを利用して.labelファイルを発行しました。

3. .binと.labelを合わせる　.lasファイルを作成しました。

4. .lasファイルをPotreeConverterで交換して.htmlファイルを作成しました。

.htmlファイルをweb上に確認しましたが、どうやってfilterからclassification（例え赤色なら建物）のをselect/unselectする時に表示・非表示できますか。学習済のrandlanet_semantickittiを利用してclassificationを分けったがこれは正しいですか。

完璧なコードを教えて

convert .e57 to .bin code

import pye57

import numpy as np

import os



def main():

    print("--- Step 1: 初期設定 ---")

    project_base_path = "/mnt/d/PointCloudLibrary/Open3D-ML"

    dataset_base_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne"

    input_e57_path = os.path.join(project_base_path, "3d_point_data/S_0612_kaisyasyuuhen_P03_001.e57")

    output_bin_path = os.path.join(dataset_base_path, "000000.bin")

    if not os.path.exists(os.path.dirname(output_bin_path)):

        os.makedirs(os.path.dirname(output_bin_path))



    print(f"--- Step 2: E57ファイルをロード ---")

    e57 = pye57.E57(input_e57_path)

    data = e57.read_scan_raw(0)

    available_keys = list(data.keys())

    print(f"[Log] 利用可能なキー: {available_keys}")

    print("--- Step 3: 座標データの抽出と変換 ---")



    # 球面座標系の場合の変換ロジック

    if 'sphericalRange' in data:

        print("[Log] 球面座標を検出しました。XYZに変換します。")

        r = data['sphericalRange'].astype(np.float32)

        az = data['sphericalAzimuth'].astype(np.float32)

        el = data['sphericalElevation'].astype(np.float32)



        # 球面座標から直交座標への変換公式

        # x = r * cos(el) * cos(az)

        # y = r * cos(el) * sin(az)

        # z = r * sin(el)

        x = r * np.cos(el) * np.cos(az)

        y = r * np.cos(el) * np.sin(az)

        z = r * np.sin(el)

        print(f"[Log] 変換完了 (点数: {len(x)})")

    # もし直交座標がある場合（念のため）

    elif 'cartesianX' in data or 'x' in data:

        x = data.get('cartesianX', data.get('x')).astype(np.float32)

        y = data.get('cartesianY', data.get('y')).astype(np.float32)

        z = data.get('cartesianZ', data.get('z')).astype(np.float32)

        print("[Log] 直交座標を抽出しました")

    else:

        print("[Error] 座標データが見つかりません。")

        return

    print("--- Step 4: 反射強度(Intensity)の抽出 ---")

    intensity = data.get('intensity', np.zeros_like(x)).astype(np.float32)

    if intensity.max() > 1.0:

        intensity /= intensity.max()

    print("--- Step 5: バイナリ保存 ---")

    kitti_points = np.column_stack((x, y, z, intensity))

    kitti_points.tofile(output_bin_path)



    print(f"--- 完了! ---")

    print(f"保存先: {output_bin_path}")



if __name__ == "__main__":



    main()



create .label using this command

ython scripts/run_pipeline.py torch \

  --pipeline SemanticSegmentation \

  --cfg_file ml3d/configs/randlanet_semantickitti.yml \

  --dataset.dataset_path /mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data \

  --dataset.name SemanticKITTI \

  --dataset.cache_dir ./logs/cache \

  --model.ckpt_path ./randlanet_semantickitti_202201071330utc.pth \

  --device gpu \

  --main_log_dir ./logs \

  --split test



convert to laz code

import numpy as np

import laspy

import os



# --- 1. パス設定 ---

bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"

label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"

output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/0b_segmented_result_vs01_17_05.laz"



# 出力先ディレクトリがない場合は作成

os.makedirs(os.path.dirname(output_laz), exist_ok=True)



# --- 2. データの読み込み ---

if not os.path.exists(bin_path):

    print(f"Error: {bin_path} が見つかりません。")

    exit()



# SemanticKITTIのバイナリ読み込み (x, y, z, intensity)

points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)

xyz = points[:, :3].astype(np.float64)



# ラベルの読み込み (下位16ビットがセマンティックラベル)

labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)



# --- 3. ボクセルダウンサンプリング (0.02 = 2cm) ---

voxel_size = 0.01

voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)

_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

# _, unique_indices = np.unique(xyz, axis=0, return_index=True)



down_xyz = xyz[unique_indices]

down_labels = labels[unique_indices]



# --- 4. LASヘッダーの設定 (Point Format 7 / Version 1.4) ---

# Format 7 は RGB と 8-bit Classification を同時にサポートします

header = laspy.LasHeader(point_format=7, version="1.4")

header.offsets = np.min(down_xyz, axis=0)

header.scales = [0.001, 0.001, 0.001]



las = laspy.LasData(header)

# las.x = down_xyz[:, 0]

# las.y = down_xyz[:, 1]

# las.z = down_xyz[:, 2]



# 代入直前にデータの型を確実に float64 にする

las.x = np.array(down_xyz[:, 0], dtype=np.float64)

las.y = np.array(down_xyz[:, 1], dtype=np.float64)

las.z = np.array(down_xyz[:, 2], dtype=np.float64)



# --- 5. Classificationのマッピング (Potreeの標準名に合わせる) ---

def map_kitti_to_asprs(labels):

    asprs_labels = np.zeros_like(labels)

    # SemanticKITTI ID -> ASPRS Standard ID

    mapping = {

        0: 0,    # unclassified

        10: 1,   # car -> Processed (ASPRS 1)

        11: 1,   # bicycle

        13: 1,   # bus

        15: 1,   # motorcycle

        18: 1,   # truck

        20: 1,   # other-vehicle

        30: 1,   # person

        40: 11,  # road -> Road (ASPRS 11)

        44: 11,  # sidewalk

        50: 6,   # building -> Building (ASPRS 6)

        51: 6,   # wall

        70: 5,   # vegetation -> High Vegetation (ASPRS 5)

        71: 4,   # trunk -> Medium Veg

        72: 3,   # terrain -> Low Veg

        80: 1,   # pole

        81: 1,   # traffic-sign

    }

    for kitti_id, asprs_id in mapping.items():

        asprs_labels[labels == kitti_id] = asprs_id

    return asprs_labels



# las.classification = map_kitti_to_asprs(down_labels).astype(np.uint8)

las.classification = down_labels.astype(np.uint8)



# --- 6. 色の割り当て (SemanticKITTI本来の色を保持) ---

# [R, G, B] (0-255)

cmap = {

    0: [100, 100, 100], 10: [100, 150, 245], 11: [100, 230, 245],

    13: [0, 0, 255],    15: [30, 60, 150],   18: [0, 0, 255],

    20: [0, 0, 255],    30: [255, 30, 30],   40: [255, 0, 255], # Road

    50: [255, 200, 0],  70: [0, 175, 0],     80: [255, 240, 150],

    81: [255, 0, 0]

}

# デフォルトは赤

colors = np.array([cmap.get(l, [255, 0, 0]) for l in down_labels])



# laspy 2.x系: uint16 (0-65535) に変換して代入

las.red = (colors[:, 0] * 256).astype(np.uint16)

las.green = (colors[:, 1] * 256).astype(np.uint16)

las.blue = (colors[:, 2] * 256).astype(np.uint16)



# --- 7. 保存 ---

las.update_header()

las.write(output_laz)



print("-" * 30)

print(f"変換成功: {output_laz}")

print(f"元の点数: {len(xyz)}")

print(f"保存点数: {len(down_xyz)}")

print("-" * 30)



index.html

<script>

    

        window.viewer = new Potree.Viewer(document.getElementById("potree_render_area"));

        

        viewer.setEDLEnabled(true);

        viewer.setFOV(60);

        viewer.setPointBudget(2_000_000);

        document.title = "";

        viewer.setEDLEnabled(false);

        viewer.setBackground("gradient"); // ["skybox", "gradient", "black", "white"];

        viewer.setDescription(``);

        viewer.loadSettingsFromURL();

        

        viewer.setDescription("");

        

        viewer.loadGUI(() => {

            viewer.setLanguage('en');

            $("#menu_appearance").next().show();

            $("#menu_tools").next().show();

            $("#menu_clipping").next().show();

            viewer.toggleSidebar();

        });

        

        Potree.loadPointCloud("pointclouds/index/cloud.js", "index", e => {

            let pointcloud = e.pointcloud;

            let material = pointcloud.material;

            viewer.scene.addPointCloud(pointcloud);

            //material.activeAttributeName = "rgba"; //[rgba, intensity, classification, ...]

            material.size = 1;

            material.pointSizeType = Potree.PointSizeType.ADAPTIVE;

            material.shape = Potree.PointShape.SQUARE;

            viewer.fitToScreen();

        });

        

    </script>

<img width="1852" height="949" alt="image" src="https://github.com/user-attachments/assets/5d456a99-0fd8-452a-b279-c61cac8c198a" />



<img width="1836" height="955" alt="image" src="https://github.com/user-attachments/assets/ad5b6367-4f7d-43bf-abb7-bf0b1050d5ba" />



<img width="1765" height="985" alt="image" src="https://github.com/user-attachments/assets/79a94227-5acc-426a-a65d-a5b0b1130578" />
