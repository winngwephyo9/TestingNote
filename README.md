import numpy as np
import laspy
import os

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/SemanticKITTI/dataset/sequences/08/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/08/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/segmented_result.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
xyz = points[:, :3]
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 数を合わせる
min_len = min(len(xyz), len(labels))
xyz, labels = xyz[:min_len], labels[:min_len]

# 2. 色情報の作成 (0-255)
def get_color_255(label):
    colors = {
        0: [128, 128, 128], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255]
    }
    return colors.get(label, [255, 255, 255])

# ここで 'rgb' 変数を定義します
rgb = np.array([get_color_255(l) for l in labels], dtype=np.uint8)

# 3. LASファイル（LAZ圧縮）の作成
# point_format=7 にすることで、RGB と Classification(255まで) が両立できます
header = laspy.LasHeader(point_format=7, version="1.4")
las = laspy.LasData(header)

# 座標の代入
las.x = xyz[:, 0]
las.y = xyz[:, 1]
las.z = xyz[:, 2]

# 分類番号の代入
las.classification = labels

# 色情報の代入 (LASは16bitカラーのため256倍)
las.red = rgb[:, 0].astype(np.uint16) * 256
las.green = rgb[:, 1].astype(np.uint16) * 256
las.blue = rgb[:, 2].astype(np.uint16) * 256

# 保存
las.write(output_laz)
print(f"--- 完了! LAZファイル(LAS 1.4)を作成しました: {output_laz} ---")


そのコードはSemanticKITTIのデータセットをlazファイルを作成して、その.lazファイルをPotreeConverterで交換してもらった.htmlファイルをweb上に表示という流れで正しくコードです。

専門点群テストデータをこのコードで確認してみると”Too many duplicate points were encountered”が発生しています。
そのため、以下のコードで変更してテストしましたが、.lazファイルがCloudCompareで正しく座標配置していますがweb上には座標は詰まっている状況になっております。
そのため、どうやって修正したらいいでしょうか。きちんと調査して修正してください。
import numpy as np
import laspy

# --- パス設定 ---
bin_path = "/mnt/d/PointCloudLibrary/data/ob_testing_point_cloud_data/dataset/sequences/00/velodyne/000000.bin"
label_path = "/mnt/d/PointCloudLibrary/Open3D-ML/test/sequences/00/predictions/000000.label"
output_laz = "/mnt/d/PointCloudLibrary/Open3D-ML/3d_point_data/ob_downsampled_final.laz"

# 1. データの読み込み
points = np.fromfile(bin_path, dtype=np.float32).reshape(-1, 4)
# xyz = points[:, :3].astype(np.float64)
xyz = points[:, :3]
labels = (np.fromfile(label_path, dtype=np.uint32) & 0xFFFF).astype(np.uint8)

# 2. ボクセルダウンサンプリング (1cm単位)
voxel_size = 0.01
voxel_indices = np.floor(xyz / voxel_size).astype(np.int64)
_, unique_indices = np.unique(voxel_indices, axis=0, return_index=True)

down_xyz = xyz[unique_indices]
down_labels = labels[unique_indices]

print(f"Original: {len(xyz)} -> Downsampled: {len(down_xyz)}")

# 3. LASヘッダーの設定 (重要: ここを修正)
header = laspy.LasHeader(point_format=7, version="1.4")

# Potree Converter 2.x系で精度を保つためのスケール (0.001 = 1mm精度)
header.scales = [0.001, 0.001, 0.001]

# 手動で引くのではなく、ヘッダーにオフセットを定義する
header.offsets = np.min(down_xyz, axis=0)

las = laspy.LasData(header)

# 座標の代入: 元の値をそのまま入れる (laspyが内部で offset を考慮して処理します)
las.x = down_xyz[:, 0]
las.y = down_xyz[:, 1]
las.z = down_xyz[:, 2]

# 4. 属性の代入
las.classification = down_labels
las.return_number = np.ones(len(down_xyz), dtype=np.uint8)
las.number_of_returns = np.ones(len(down_xyz), dtype=np.uint8)

# 5. 色の割り当て (16bitカラー対応)
def get_color_255(label):
    colors = {
        0: [100, 100, 100], 10: [0, 0, 255], 11: [77, 179, 255],
        13: [0, 77, 128], 15: [255, 128, 0], 18: [128, 51, 26],
        20: [77, 77, 153], 30: [255, 51, 51], 40: [255, 0, 255],
        50: [255, 204, 0], 70: [0, 255, 0], 80: [0, 255, 255], 81: [255, 255, 0]
    }
    return colors.get(label, [255, 0, 0]) 

rgb = np.array([get_color_255(l) for l in down_labels], dtype=np.uint16)
# LASは各チャンネル16bit(0-65535)なので、256倍する
las.red = rgb[:, 0] * 256
las.green = rgb[:, 1] * 256
las.blue = rgb[:, 2] * 256

# 6. 保存
las.update_header()
las.write(output_laz)
print(f"--- 完了! {output_laz} ---")

SemanticKITTIのテストデータセットの正しく表示
<img width="1368" height="1002" alt="image" src="https://github.com/user-attachments/assets/76a0526f-6e57-4ec4-b54e-2b505167b636" />
現在のテストデータの詰まって状態
<img width="1107" height="990" alt="image" src="https://github.com/user-attachments/assets/9a05170c-2135-4f4c-9dff-11821ee47cad" />


