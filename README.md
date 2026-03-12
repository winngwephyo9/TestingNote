import numpy as np
import laspy
import os

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/ob_downsampled_final_11.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3].astype(np.float64)
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 2. ボクセルダウンサンプリング (Web表示用に少し粗めに設定 2cm 推奨)
voxel_size = 0.05
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

# 3. LASヘッダーの設定 (ここがWeb表示の鍵)
header = laspy.LasHeader(point_format=7, version="1.4")
header.scales = [0.001, 0.001, 0.001] # 1mm精度

# オフセットの設定（最小値から少し引いて安全マージンを確保）
header.offsets = np.min(down_xyz, axis=0) - 0.1
# 【重要】オフセットをデータの最小値に設定することで、
# Potree内部での計算精度を最大化します。
# header.offsets = np.min(down_xyz, axis=0)
# header.scales = [0.0001, 0.0001, 0.0001] # 0.1mm精度に上げる
# header.offsets = np.mean(down_xyz, axis=0) # 最小値ではなく「平均値」を中央にする

las = laspy.LasData(header)
# laspyはヘッダーのオフセットを設定していれば、
# 以下の代入時に自動で「(値 - オフセット) / スケール」を計算して保存します。
las.x = down_xyz[:, 0] 
las.y = down_xyz[:, 1] 
las.z = down_xyz[:, 2] 

# 【最重要】データの統計情報をヘッダーに書き込む（これがないとWebで映らない）
las.update_header()

# 4. 属性の代入
# las.classification = down_labels
las.classification = down_labels.astype(np.uint8)

# 5. 色の割り当て (高速化版)
def get_color_map():
    return {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }

cmap = get_color_map()
# デフォルト色[255,0,0]を適用しつつマッピング
rgb = np.array([cmap.get(l, [255, 0, 0]) for l in down_labels], dtype=np.uint16)

# LASは16bitカラー(0-65535)を必要とするため256倍する
# las.red = rgb[:, 0] * 256
# las.green = rgb[:, 1] * 256
# las.blue = rgb[:, 2] * 256
las.red = (rgb[:, 0] * 256).astype(np.uint16)
las.green = (rgb[:, 1] * 256).astype(np.uint16)
las.blue = (rgb[:, 2] * 256).astype(np.uint16)

# 6. 保存
las.write(output_laz)
print(f"Original points: {len(xyz)} -> Downsampled: {len(down_xyz)}")
print(f"Offsets used: {header.offsets}")

このようにコードを実行して、.lasファイルを作成した後web上に確認してみると形が正しく表示できない状況になっています。
修正してください。.lasファイルをPotreeConverterで交換してweb上に確認しましたがCloudCompareに表示している点群みたいの形になっていないため、どのように修正したらいいでしょうか。
そして、classificationは画像の通りになっているので実際の時はroad,などclassificationがある可能性があると思ういます。
<img width="1529" height="750" alt="image" src="https://github.com/user-attachments/assets/53390207-d29c-483f-8f4c-dd1bf73688d8" />
<img width="1492" height="785" alt="image" src="https://github.com/user-attachments/assets/6e685baa-d1a0-4472-864b-02ff04009b62" />
<img width="1172" height="950" alt="image" src="https://github.com/user-attachments/assets/d4e376a4-05c5-426f-9756-6ba49eec606d" />



